---
author: Carlos Mendible
categories:
- dotnet
- dotnetcore
crosspost_to_medium: true
date: "2017-05-08T10:55:32Z"
description: 'Step by step: Kafka Pub/Sub with Docker and .Net Core'
image: /wp-content/uploads/2017/05/kafka-logo-wide.png
tags: ["Docker", "kafka", "pubsub"]
title: 'Step by step: Kafka Pub/Sub with Docker and .Net Core'
url: /2017/05/08/step-by-step-kafka-pub-sub-with-docker-and-net-core/
---
Last week I attended to a <a href="https://kafka.apache.org/" target="_blank">Kafka</a> workshop and this is my attempt to show you a simple **Step by step: Kafka Pub/Sub with Docker and .Net Core** tutorial.

Let's start:

## 1. Create a folder for your new project
---
Open a command prompt an run 
    
``` powershell
mkdir kafka.pubsub.console
cd kafka.pubsub.console
```

## 2. Create a console project
---
``` powershell
dotnet new console
```

## 3. Add the Confluent.Kafka nuget package
---
Add the **Confluent.Kafka** nuget package to your project:
    
``` powershell
dotnet add package Confluent.Kafka -v 0.9.5
dotnet restore
```

## 4. Replace the contents of Program.cs
---

Replace the contents of the **Program.cs** file with the following code:

    
``` csharp
using System;
using System.Collections.Generic;
using System.Text;
using System.Threading.Tasks;
using Confluent.Kafka;
using Confluent.Kafka.Serialization;

namespace kafka.pubsub.console
{
    class Program
    {
        static void Main(string[] args)
        {
            // The Kafka endpoint address
            string kafkaEndpoint = "127.0.0.1:9092";

            // The Kafka topic we'll be using
            string kafkaTopic = "testtopic";

            // Create the producer configuration
            var producerConfig = new Dictionary<string, object> { { "bootstrap.servers", kafkaEndpoint } };

            // Create the producer
            using (var producer = new Producer<Null, string>(producerConfig, null, new StringSerializer(Encoding.UTF8)))
            {
                // Send 10 messages to the topic
                for (int i = 0; i < 10; i++)
                {
                    var message = $"Event {i}";
                    var result = producer.ProduceAsync(kafkaTopic, null, message).GetAwaiter().GetResult();
                    Console.WriteLine($"Event {i} sent on Partition: {result.Partition} with Offset: {result.Offset}");
                }
            }

            // Create the consumer configuration
            var consumerConfig = new Dictionary<string, object>
            {
                { "group.id", "myconsumer" },
                { "bootstrap.servers", kafkaEndpoint },
            };

            // Create the consumer
            using (var consumer = new Consumer<Null, string>(consumerConfig, null, new StringDeserializer(Encoding.UTF8)))
            {
                // Subscribe to the OnMessage event
                consumer.OnMessage += (obj, msg) =>
                {
                    Console.WriteLine($"Received: {msg.Value}");
                };

                // Subscribe to the Kafka topic
                consumer.Subscribe(new List<string>() { kafkaTopic });

                // Handle Cancel Keypress 
                var cancelled = false;
                Console.CancelKeyPress += (_, e) =>
                {
                    e.Cancel = true; // prevent the process from terminating.
                    cancelled = true;
                };

                Console.WriteLine("Ctrl-C to exit.");

                // Poll for messages
                while (!cancelled)
                {
                    consumer.Poll();
                }
            }
        }
    }
}
```

## 5. Start Kafka with Docker
---

You'll need to add the following address range to your **docker unsafe registry**: **172.18.0.0/16**
Create a **docker-compose.yml** file with the following contents:

    
``` yml
version: '2'
services:
  zookeeper:
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"
  kafka:
    image: wurstmeister/kafka:0.10.2.0
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: 127.0.0.1
      KAFKA_CREATE_TOPICS: "testtopic:1:1"
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

And run the following command:

    
``` powershell
docker-compose up
```
    
It will take a while but you'll get a working Kafka installation.
      
## 6. Run the program
---
Run the program and enjoy!
          
``` powershell
dotnet run
```
      
Get the code and related files here: <a href="https://github.com/cmendible/dotnetcore.samples/tree/master/kafka.pubsub.console"  target="_blank">https://github.com/cmendible/dotnetcore.samples/tree/master/kafka.pubsub.console</a>

Hope it helps!  