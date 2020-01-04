---
author: Carlos Mendible
categories:
- Azure
- dotNetCore
date: "2016-07-17T15:21:03Z"
description: 'Step by step: .NET Core, Azure Service Bus and AMQP'
image: /wp-content/uploads/2016/02/Microsoft-Service-Bus-logo.png
tags: ["AMQP", "ServiceBus"]
title: 'Step by step: .NET Core, Azure Service Bus and AMQP'
url: /2016/07/17/step-by-step-net-core-azure-service-bus-and-amqp/
---
Today I'll show you how to create a small console application with a **Step by step: .NET Core, Azure Service Bus and AMQP** example.

First be aware of the following prerequisites:

<table>
    <tr>
      <td>
        **OS**
      </td>
      <td>
        **Prerequisites**
      </td>
    </tr>
    <tr>
      <td>
        Windows
      </td>
      <td>
        Windows: You must have <a href="https://go.microsoft.com/fwlink/?LinkID=809122" target="_blank">.NET Core SDK for Windows</a> or both <a href="https://www.visualstudio.com/news/releasenotes/vs2015-update3-vs" target="_blank">Visual Studio 2015 Update 3*</a> and <a href="https://go.microsoft.com/fwlink/?LinkId=817245" target="_blank">.NET Core 1.0 for Visual Studio</a> installed.
      </td>
    </tr>
    <tr>
      <td>
        linux, mac or docker
      </td>
      <td>
        checkout <a href="https://www.microsoft.com/net/" target="_blank">.NET Core</a>
      </td>
    </tr>
</table>

You will also need an Azure Service Bus namespace, topic and subscription with the correct set of credentials. (Start here: <a href="https://azure.microsoft.com/en-us/documentation/articles/service-bus-dotnet-how-to-use-topics-subscriptions/" target="_blank">How to use Service Bus topics and subscriptions</a> if you don't know how to configure the Service Bus)

Now let's start:

## 1. Create a folder for your new project
---

Open a command promt an run 
    
``` powershell
mkdir azureservicebus.amqp.console
```
## 2. Create the project
---

``` powershell
cd azureservicebus.amqp.console
dotnet new
```

## 3. Create a settings file
---
Create an **appsettings.json** file to hold your service bus connection information (remember to replace the values with those from your Service Bus configuration):

``` json
{
    "ServiceBus": {
        "NamespaceUrl": "[The namespace url (i.e. codeityourself.servicebus.windows.net)]",
        "PolicyName": "[The policy name]",
        "Key": "[The SAS key]"
    }
}
```

## 4. Modify the project file
---
Modify the **project.json** to add the **AMQPNetLite** dependencies and also specify that the **appsettings.json** file must be copied to the output (**buildOptions** section) so it becomes available to the application once you build it.
    
``` json
{
  "version": "1.0.0-*",
  "buildOptions": {
    "debugType": "portable",
    "emitEntryPoint": true,
     "copyToOutput": {
      "include": "appsettings.json"
    }
  },
  "dependencies": {
    "Microsoft.Extensions.Configuration": "1.0.0",
    "Microsoft.Extensions.Configuration.Json": "1.0.0",
    "AMQPNetLite": "1.2.0",
    "System.Net.Http": "4.1.0",
    "System.Xml.XmlDocument": "4.0.1",
    "System.Runtime.Serialization.Xml": "4.1.1"
  },
  "frameworks": {
    "netcoreapp1.0": {
      "dependencies": {
        "Microsoft.NETCore.App": {
          "type": "platform",
          "version": "1.0.0"
        }
      },
      "imports": "dnxcore50"
    }
  }
}
```

## 5. Restore packages
---
You just modified the **project.json** file with new dependencies so please restore the packages with the following command:

``` powershell
dotnet restore
```
## 6. Modify Program.cs
---
Replace the contents of the **Program.cs** file with the following code 
    
``` csharp
namespace ConsoleApplication
{
    using System;
    using System.IO;
    using System.Net;
    using System.Xml;
    using Amqp;
    using Amqp.Framing;
    using Microsoft.Extensions.Configuration;

    public class Program
    {
        public static void Main(string[] args)
        {
            // Enable to app to read json setting files
            var builder = new ConfigurationBuilder()
                .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true);

            // build the configuration
            var configuration = builder.Build();

            // Azure service bus SAS key
            var policyName = WebUtility.UrlEncode(configuration["ServiceBus:PolicyName"]);
            var key = WebUtility.UrlEncode(configuration["ServiceBus:Key"]);

            // Azure service bus namespace
            var namespaceUrl = configuration["ServiceBus:NamespaceUrl"];

            // Create the AMQP connection string
            var connnectionString = $"amqps://{policyName}:{key}@{namespaceUrl}/";

            // Create the AMQP connection
            var connection = new Connection(new Address(connnectionString));

            // Create the AMQP session
            var amqpSession = new Session(connection);

            // Give a name to the sender
            var senderSubscriptionId = "codeityourself.amqp.sender";
            // Give a name to the receiver
            var receiverSubscriptionId = "codeityourself.amqp.receiver";

            // Name of the topic you will be sending messages
            var topic = "codeityourself";

            // Name of the subscription you will receive messages from
            var subscription = "codeityourself.listener";

            // Create the AMQP sender
            var sender = new SenderLink(amqpSession, senderSubscriptionId, topic);

            for (var i = 0; i < 10; i++)
            {
                // Create message
                var message = new Message($"Received message {i}");

                // Add a meesage id
                message.Properties = new Properties() { MessageId = Guid.NewGuid().ToString() };

                // Add some message properties
                message.ApplicationProperties = new ApplicationProperties();
                message.ApplicationProperties["Message.Type.FullName"] = typeof(string).FullName;

                // Send message
                sender.Send(message);
            }

            // Create the AMQP consumer
            var consumer = new ReceiverLink(amqpSession, receiverSubscriptionId, $"{topic}/Subscriptions/{subscription}");

            // Start listening
            consumer.Start(5, OnMessageCallback);

            // Wait for a key to close the program
            Console.Read();
        }

        /// <summary>
        /// Method that will be called every time a message is received.
        /// </summary>
        /// 
        /// 
        static void OnMessageCallback(ReceiverLink receiver, Message message)
        {
            try
            {
                // You can read the custom property
                var messageType = message.ApplicationProperties["Message.Type.FullName"];

                // Variable to save the body of the message.
                string body = string.Empty;

                // Get the body
                var rawBody = message.GetBody<object>();
  
                  // If the body is byte[] assume it was sent as a BrokeredMessage  
                  // and deserialize it using a XmlDictionaryReader
                  if (rawBody is byte[])
                  {
                      using (var reader = XmlDictionaryReader.CreateBinaryReader(
                          new MemoryStream(rawBody as byte[]),
                          null,
                          XmlDictionaryReaderQuotas.Max))
                      {
                          var doc = new XmlDocument();
                          doc.Load(reader);
                          body = doc.InnerText;
                      }
                  }
                  else // Asume the body is a string
                  {
                      body = rawBody.ToString();
                  }
  
                  // Write the body to the Console.
                  Console.WriteLine(body);
  
                  // Accept the messsage.
                  receiver.Accept(message);
              }
              catch (Exception ex)
              {
                  receiver.Reject(message);
                  Console.WriteLine(ex);
              }
          }
      }
  }
  ```

## 8. Build
---  
Build the application with the following command

``` powershell
dotnet build
```

# 9. Run
---  
    
You are good to go so run the application
      
``` powershell
dotnet run
```

You can get the code here: <a href="https://github.com/cmendible/dotnetcore.samples/tree/master/azureservicebus.amqp.console">https://github.com/cmendible/dotnetcore.samples/tree/master/azureservicebus.amqp.console</a>

Hope it helps!