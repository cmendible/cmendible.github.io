---
author: Carlos Mendible
categories:
- azure
- dotnetcore
crosspost_to_medium: true
date: "2017-08-04T08:50:13Z"
description: Read Azure Service Health Activity Logs with .NET Core
image: /wp-content/uploads/2017/08/loganalytics.png
# tags: ActivityLogs HealthEndpoint LogAnalytics ServiceHealth
title: Read Azure Service Health Activity Logs with .NET Core
---
This week there was a small outage within the Azure Data Lake Store service and as consequence I wondered how could I **Read Azure Service Health Activity Logs with .NET Core.**

Let's go for it:

## 1. Create a folder for your new project
---
Open a command promt an run: 
    
``` powershell
mkdir azure.health
```
## 2. Create the project
---
``` powershell
cd azure.health
dotnet new console
```
## 3. Add the references needed to query Azure Health Events
---
``` powershell
dotnet add package Microsoft.Azure.Insights -v 0.15.0-preview
dotnet add package Microsoft.Azure.Management.Fluent
dotnet restore
```

<a href="https://www.nuget.org/packages/Microsoft.Azure.Insights/" target="_blank">Microsoft.Azure.Insights</a> will let you query the events and <a href="https://www.nuget.org/packages/Microsoft.Azure.Management.Fluent" target="_blank">Microsoft.Azure.Management.Fluent</a> will let the application authenticate to Azure

## 4. Replace the content of Program.cs
---
Replace the content of Program.cs with the following contents:
          
``` csharp
/// <summary>
/// Query Azure.Health Events using .NET Core
/// The code was inspired on: How to Retrieve Azure Service Health Event Logs (https://code.msdn.microsoft.com/windowsapps/How-To-Programmatically-49df487d/view/Reviews)
/// by Matt Loflin
/// </summary>
namespace azure.health
{
    using System;
    using System.Linq;
    using Microsoft.Azure.Insights;
    using Microsoft.Azure.Insights.Models;
    using Microsoft.Azure.Management.Fluent;
    using Microsoft.Azure.Management.ResourceManager.Fluent;
    using Microsoft.Rest.Azure.OData;

    class Program
    {
        static void Main(string[] args)
        {
            // The file with the Azure Service Principal Credentials.
            var authFile = "my.azureauth";

            // Parse the credentials from the file.
            var credentials = SdkContext.AzureCredentialsFactory.FromFile(authFile);

            // Authenticate with Azure
            var azure = Azure
                .Configure()
                .Authenticate(credentials)
                .WithDefaultSubscription();

            // Create an InsightsClient instance.
            var client = new InsightsClient(credentials);

            // If we subscription is not set the API call will fail. 
            client.SubscriptionId = credentials.DefaultSubscriptionId;

            // Create the OData filter for a time interval and the Azure.Health Provider.
            // Search back one day.
            var days = -1;
            var endDateTime = DateTime.Now;
            var startDateTime = endDateTime.AddDays(days);
            string filterString = FilterString.Generate<EventDataForFilter>(eventData =>
                (eventData.EventTimestamp >= startDateTime) &&
                (eventData.EventTimestamp <= endDateTime) &#038;&#038;
                (eventData.ResourceProvider == "Azure.Health"));

            // Get the Events from Azure.
            var response = client.Events.List(filterString);
            while (response != null &#038;&#038; response.Any())
            {
                foreach (var eventData in response)
                {
                    // Set the Console Color according to the Event Status. 
                    if (eventData.Status.Value != "Resolved" &#038;&#038;
                        eventData.Level <= EventLevel.Warning)
                    {
                        Console.ForegroundColor = ConsoleColor.Red;
                    }
                    else if (eventData.Status.Value == "Resolved")
                    {
                        Console.ForegroundColor = ConsoleColor.Green;
                    }
                    else
                    {
                        Console.ForegroundColor = ConsoleColor.White;
                    }

                    // Write event data to the console
                    Console.WriteLine($"{eventData.EventTimestamp.ToLocalTime()} - {eventData.ResourceProviderName.Value} - {eventData.OperationName.Value}");
                    Console.WriteLine($"Status:\t {eventData.Status.Value}");
                    Console.WriteLine($"Level:\t {eventData.Level.ToString()}");
                    Console.WriteLine($"CorrelationId:\t {eventData.CorrelationId}");
                    Console.WriteLine($"Resource Type:\t {eventData.ResourceType.Value}");
                    Console.WriteLine($"Description:\t {eventData.Description}");
                }

                // Get more events if available.
                if (!string.IsNullOrEmpty(response.NextPageLink))
                {
                    response = client.Events.ListNext(response.NextPageLink);
                }
                else
                {
                    response = null;
                }
            }

            Console.ForegroundColor = ConsoleColor.Green;
            Console.WriteLine("No more events...");
            Console.ForegroundColor = ConsoleColor.White;
        }
    }

    // EventData Filter. 
    public class EventDataForFilter
    {
        /// <summary>
        /// Event Timestamp
        /// </summary>
        public DateTime EventTimestamp { get; set; }

        /// <summary>
        /// Resource Provider
        /// </summary>
        public string ResourceProvider { get; set; }
    }
}
```

## 5. Create the my.azureauth file
---
To authenticate you'll need a my.azureauth file with the following format:

``` xml
subscription=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
client=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
tenant=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
key=XXXXXX
managementURI=https\://management.core.windows.net/
baseURL=https\://management.azure.com/
authURL=https\://login.windows.net/
https\://graph.windows.net/
```
          
As you can see you'll be needing your **Azure Subscription Id**, **Tenand Id** and a valid **<a href="https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-authenticate-service-principal" target="_blank">Service Principal</a>** and **Key**.

If you check my previous post: <a href="https://carlos.mendible.com/2017/08/02/create-service-principal-write-required-parameters-to-azureauth-file/">Create a Service Principal and write required parameters to a .azureauth file</a> and run the script you'll get a file with the credentials. 
            
## 6. Run the application
---
Run the following command:
                
  ``` powershell
dotnet run
```
You can get the code <a href="https://github.com/cmendible/dotnetcore.samples/tree/master/azure.health" target="_blank">here</a>.

Hope it helps!