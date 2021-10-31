---
author: Carlos Mendible
categories:
- azure
- dotnet
date: "2016-03-13T14:39:25Z"
description: 'EasyAzureServiceBus: easy Service Bus 1.1 for Windows Server'
images: ["/wp-content/uploads/2016/02/Microsoft-Service-Bus-logo.png"]
tags: ["service bus"]
title: 'EasyAzureServiceBus: easy Service Bus 1.1 for Windows Server'
url: /2016/03/13/easyazureservicebus-easy-service-bus-1-1-windows-server/
---
A couple of years ago I started to work with one of my clients to implement an on-premises <a href="https://en.wikipedia.org/wiki/Enterprise_service_bus" target="_blank">Service Bus</a> solution.

While investigating the options I discovered a nice library, <a href="http://easynetq.com" target="_blank">EasyNetQ</a>: an easy .NET API for RabbitMQ, which inspired me to create a very similar project to simplify the use of <a href="https://en.wikipedia.org/wiki/Publishâ€“subscribe_pattern" target="_blank">pub/sub</a> messaging with <a href="https://msdn.microsoft.com/en-us/library/dn282144.aspx" target="_blank">Service Bus 1.1 for Windows Server</a>.

I named the project <a style="font-weight: bold;" href="https://github.com/cmendible/EasyAzureServiceBus" target="_blank">EasyAzureServiceBus</a> and just as <a href="http://easynetq.com" target="_blank">EasyNetQ</a>, it's very simple to use and available to all of you as an open source project:


To connect:

``` csharp
// Create a bus instance
IBus bus = AzureCloud.CreateBus();
```

To publish:

``` csharp
// Publish a message
bus.Publish(message);
``` 


To suscribe:

``` csharp
// Subscribe to a message of type Message
bus.Subscribe("subscriptionId", msg => Console.WriteLine(msg.Text));
``` 
  
To install:

``` powershell
PM> Install-Package EasyAzureServiceBus
``` 

While the project already supports queues and the nice auto-subscribe feature,  it is far from being perfect, and therefor I'm planning for the following features in the near future:

  * Add logging capabilities
  * Proper error handling
  * Improve configuration options, such as: ReceivedMode, Autocomplete, Max Concurrent Calls and Duplicate detection.
  * Extend compatibility to <a href="https://azure.microsoft.com/en-us/documentation/articles/service-bus-fundamentals-hybrid-solutions/" target="_blank">Microsoft Azure Service Bus</a> &#8211;> the real deal isn't it?

Hope it helps.