---
author: Carlos Mendible
categories:
- azure
crosspost_to_medium: true
date: "2018-02-18T17:00:00Z"
description: Consuming Azure Event Hubs Events With Kafka Connect
image: /assets/img/posts/azureventhubs.png
published: true
# tags: kafka kafkaconnect eventhubs
title: Consuming Azure Event Hubs Events With Kafka Connect
---

So last week I was in a rush to find a fast and easy way to consume events from Azure Event Hubs and send them to a Kafka topic.

After googling a bit I found this project: [Kafka Connect Azure IoT Hub](https://github.com/Azure/toketi-kafka-connect-iothub/releases). Yes the name of the project can be misleading, but since IoT Hub is a service which relies on Event Hubs and also taking a close look to the code showed that it uses the Event Hubs client for java, I decided to give it a try.

I will asume not only that you have working knowledge with Event Hubs, but also that you have an instance deployed, plus a working Kafka and Kafka Connect (Distributed mode) setup.

Let's kick it:

# 1. Download the Kafka Connect Azure IoT Hub

Download the [Kafka Connect Azure IoT Hub 0.6 jar](https://github.com/Azure/toketi-kafka-connect-iothub/releases/tag/v0.6) and copy the file in the Kafka installation libs folder (usually under KAFKA_HOME/libs).

Be sure to start Zookeper, Kafka and Kafka connect.

# 2. Create a connect-eventhub-source.json file

 Update the following json and save it as **connect-eventhub-source.json**.

``` json
{
    "name": "eventhub-source",
    "config": {
        "connector.class": "com.microsoft.azure.iot.kafka.connect.IotHubSourceConnector",
        "tasks.max": "[Number of task == Number Event Hub Partitions]",
        "Kafka.Topic": "[Target Kafka Topic]",
        "IotHub.EventHubCompatibleName": "[Event Hubs Name]",
        "IotHub.EventHubCompatibleEndpoint": "sb://[Event Hubs Namespace].servicebus.windows.net/",
        "IotHub.AccessKeyName": "[Access key name for the Event Hub]",
        "IotHub.AccessKeyValue": "[Access key value for the Event Hub]",
        "IotHub.ConsumerGroup": "[Consumer group (Can use $Default)]",
        "IotHub.Partitions": "[Number of Event Hub Partitions]",
        "IotHub.StartTime": "",
        "IotHub.Offsets": "",
        "BatchSize": "100"
    }
}
```

I've submited a [pull request](https://github.com/Azure/toketi-kafka-connect-iothub/pull/16) to fix some of the descriptions you'll find for the fields [here](https://github.com/Azure/toketi-kafka-connect-iothub/blob/master/README_Source.md)

# 3. Post the configuration to the Kafka Connect endpoint

Assuming your Kafka Connect is running on **localhost** and listening to the default port **8083** execute the followinmg command:

``` bash
curl -H "Content-Type: application/json" -d @connect-eventhub-source.json -X POST http://127.0.0.1:8083/connectors
```

After a while you should start receiving the events in the Kafka Topic you configured.

Hope it helps!