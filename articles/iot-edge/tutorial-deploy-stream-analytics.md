---
# Mandatory fields. See more on aka.ms/skyeye/meta.
title: Deploy Azure Stream Analytics with Azure IoT Edge | Microsoft Docs 
description: Deploy Azure Stream Analytics as a module to an edge device
services: iot-edge
keywords: 
author: msebolt
manager: timlt

ms.author: v-masebo
ms.date: 11/28/2017
ms.topic: article
ms.service: iot-edge

# Optional fields. Don't forget to remove # if you need a field.
# ms.custom: can-be-multiple-comma-separated
# ms.devlang:devlang-from-white-list
# ms.suite: 
# ms.tgt_pltfrm:
# ms.reviewer:
---

# Deploy Azure Stream Analytics as an IoT Edge module - preview

IoT devices can produce large quantities of data. Sometimes this data has to be analyzed or processed before reaching the cloud to reduce the size of uploaded data or to eliminate the round-trip latency of an actionable insight.

IoT Edge takes advantage of pre-built Azure Service IoT Edge modules for quick deployment and [Azure Stream Analytics][azure-stream] (ASA) is one such module. You can create an ASA job from its portal, then come to IoT Hub portal to deploy it as an IoT Edge Module.  

Azure Stream Analytics provides a richly structured query syntax for data analysis both in the cloud and on IoT Edge devices. For more information about ASA on IoT Edge, see [ASA documentation](../stream-analytics/stream-analytics-edge.md).

This tutorial walks you through the creation of an Azure Stream Analytics job, and its deployment on an IoT Edge device in order to process a local telemetry stream directly on the device, and generate alerts to drive immediate action on the device.  There are two modules involved in this tutorial. A simulated temperature sensor module (tempSensor) generates temperature data from 20 to 120 degrees, incremented by 1 every 5 seconds. A Stream Analytics module resets the tempSensor when the 30 seconds average reaches 70. In a production environment, this functionality might be used to shut off a machine or take preventative measures when the temperature reaches dangerous levels. 

You learn how to:

> [!div class="checklist"]
> * Create an ASA job to process data on the Edge
> * Connect the new ASA job with other IoT Edge modules
> * Deploy the ASA job to an IoT Edge device

## Prerequisites

* An IoT Hub 
* The device that you created and configured in the quickstart or in Deploy Azure IoT Edge om a simulated device in [Windows][lnk-tutorial1-win] and [Linux][lnk-tutorial1-lin]. You need to know the device connection key and the device ID. 
* Docker running on your IoT Edge device
    * [Install Docker on Windows][lnk-docker-windows]
    * [Install Docker on Linux][lnk-docker-linux]
* Python 2.7.x on your IoT Edge device
    * [Install Python 2.7 on Windows][lnk-python].
    * Most Linux distributions, including Ubuntu, already have Python 2.7 installed.  Use the following command to make sure pip is installed: `sudo apt-get install python-pip`.


## Create an ASA job

In this section, you create an Azure Stream Analytics job to take data from your IoT hub, query the sent telemetry data from your device, and forward the results to an Azure Storage Container (BLOB). For more information, see the **Overview** section of the [Stream Analytics Documentation][azure-stream]. 

### Create a storage account

An Azure Storage account is required to provide an endpoint to be used as an output in your ASA job. The example below uses the BLOB storage type.  For more information, see the **Blobs** section of the [Azure Storage Documentation][azure-storage].

1. In the Azure portal, navigate to **Create a resource** and enter `Storage account` in the search bar. Select **Storage account - blob, file, table, queue**.

2. Enter a name for your storage account, and select the same location where your IoT hub is located. Click **Create**. Remember the name for later.

    ![new storage account][1]

3. Navigate to the storage account that you just created. Click **Browse blobs**. 
4. Create a new container for the ASA module to store data. Set the access level to **Container**. Click **OK**.

    ![storage settings][10]

### Create a Stream Analytics job

1. In the Azure portal, navigate to **Create a resource** > **Internet of Things** and select **Stream Analytics Job**.

2. Enter a name, choose **Edge** as the Hosting environment, and use the remaining default values.  Click **Create**.

    >[!NOTE]
    >Currently, ASA jobs on IoT Edge aren't supported in the West US 2 region. 

3. Go into the created job. Select **Inputs** then click **Add**.

4. For the input alias enter `temperature`, set the source type to **Data stream**, and use defaults for the other parameters. Click **Create**.

   ![ASA input](./media/tutorial-deploy-stream-analytics/asa_input.png)

5. Select **Outputs** then click **Add**.

6. For the output alias enter `alert`, and use defaults for the other parameters. Click **Create**.

   ![ASA output](./media/tutorial-deploy-stream-analytics/asa_output.png)


7. Select **Query**.
8. Replace the default text with the following query:

    ```sql
    SELECT  
        'reset' AS command 
    INTO 
       alert 
    FROM 
       temperature TIMESTAMP BY timeCreated 
    GROUP BY TumblingWindow(second,30) 
    HAVING Avg(machine.temperature) > 70
    ```
9. Click **Save**.

## Deploy the job

You are now ready to deploy the ASA job on your IoT Edge device.

1. In the Azure portal, in your IoT hub, navigate to **IoT Edge (preview)** and open the details page for your IoT Edge device.
1. Select **Set modules**.
1. If you previously deployed the tempSensor module on this device, it may autopopulate. If not, use the following steps to add that module:
   1. Click **Add IoT Edge Module**
   1. Enter `tempSensor` as the name, and `microsoft/azureiotedge-simulated-temperature-sensor:1.0-preview` for the Image URI. 
   1. Leave the other settings unchanged, and click **Save**.
1. To add your ASA Edge job, select **Import Azure Stream Analytics IoT Edge Module**.
1. Select your subscription and the ASA Edge job that you created. 
1. Select your subscription and the storage account that you created. Click **Save**.

    ![set module][6]

1. Copy the name that was automatically generated for your ASA module. 

    ![temperature module][11]

1. Click **Next** to configure routes.
1. Copy the following to **Routes**.  Replace _{moduleName}_ with the module name that you copied:

    ```json
    {
        "routes": {                                                               
          "telemetryToCloud": "FROM /messages/modules/tempSensor/* INTO $upstream", 
          "alertsToCloud": "FROM /messages/modules/{moduleName}/* INTO $upstream", 
          "alertsToReset": "FROM /messages/modules/{moduleName}/* INTO BrokeredEndpoint(\"/modules/tempSensor/inputs/control\")", 
          "telemetryToAsa": "FROM /messages/modules/tempSensor/* INTO BrokeredEndpoint(\"/modules/{moduleName}/inputs/temperature\")" 
        }
    }
    ```

1. Click **Next**.

1. In the **Review Template** step, click **Submit**.

1. Return to the device details page and click **Refresh**.  You should see the new Stream Analytics module running along with the **IoT Edge agent** module and the **IoT Edge hub**.

    ![module output][7]

## View data

Now you can go to your IoT Edge device to check out the interaction between the ASA module and the tempSensor module.

Check that all the modules are running in Docker:

   ```cmd/sh
   docker ps  
   ```

   ![docker output][8]

See all system logs and metrics data. Use the Stream Analytics module name:

   ```cmd/sh
   docker logs -f {moduleName}  
   ```

You should be able to watch the machine's temperature gradually rise until it reaches 70 degrees for 30 seconds. Then the Stream Analytics module triggers a reset and the machine temperature drops back to 21. 

   ![docker log][9]


## Next steps

In this tutorial, you configured an Azure Storage container and a Streaming Analytics job to analyze data from your IoT Edge device.  You then loaded a custom ASA module to move data from your device, through the stream, into a BLOB for download.  You can continue on to other tutorials to further see how Azure IoT Edge can create solutions for your business.

> [!div class="nextstepaction"] 
> [Deploy an Azure Machine Learning model as a module][lnk-ml-tutorial]

<!-- Images. -->
[1]: ./media/tutorial-deploy-stream-analytics/storage.png
[4]: ./media/tutorial-deploy-stream-analytics/add_device.png
[5]: ./media/tutorial-deploy-stream-analytics/asa_job.png
[6]: ./media/tutorial-deploy-stream-analytics/set_module.png
[7]: ./media/tutorial-deploy-stream-analytics/module_output.png
[8]: ./media/tutorial-deploy-stream-analytics/docker_output.png
[9]: ./media/tutorial-deploy-stream-analytics/docker_log.png
[10]: ./media/tutorial-deploy-stream-analytics/storage_settings.png
[11]: ./media/tutorial-deploy-stream-analytics/temp_module.png


<!-- Links -->
[lnk-what-is-iot-edge]: what-is-iot-edge.md
[lnk-module-dev]: module-development.md
[iot-hub-get-started-create-hub]: ../../includes/iot-hub-get-started-create-hub.md
[azure-iot]: https://docs.microsoft.com/en-us/azure/iot-hub/
[azure-storage]: https://docs.microsoft.com/en-us/azure/storage/
[azure-stream]: https://docs.microsoft.com/en-us/azure/stream-analytics/
[lnk-free-trial]: http://azure.microsoft.com/pricing/free-trial/
[lnk-tutorial1-win]: tutorial-simulate-device-windows.md
[lnk-tutorial1-lin]: tutorial-simulate-device-linux.md
[lnk-module-tutorial]: tutorial-csharp-module.md
[lnk-ml-tutorial]: tutorial-deploy-machine-learning.md

[lnk-docker-windows]: https://docs.docker.com/docker-for-windows/install/ 
[lnk-docker-linux]: https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/
[lnk-python]: https://www.python.org/downloads/