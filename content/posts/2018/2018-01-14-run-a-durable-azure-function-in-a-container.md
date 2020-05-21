---
author: Carlos Mendible
categories:
- azure
- dotnet
crosspost_to_medium: true
date: "2018-01-14T15:19:00Z"
description: Run a Durable Azure Function in a Container
images: ["/assets/img/posts/azurefunctions.jpg"]
published: true
tags: ["Docker", "AzureFunctions", "Dockerfile", "Serverless"]
title: Run a Durable Azure Function in a Container
---

Greetings readers! Hope you all a Happy New Year!

Last post I was about [running a Precompiled .NET Core Azure Function in a Container](https://carlos.mendible.com/2017/12/28/run-a-precomplied-net-core-azure-function-in-a-container). This time let's go one step further and **Run a Durable Azure Function in a Container**

**Prerequisites**:

* [Docker](https://www.docker.com) installed and basic knowledge.
* [.NET Core](https://www.microsoft.com/net/download/)
* [Azure Storage Account](https://docs.microsoft.com/en-us/azure/storage/common/storage-create-storage-account)
* [Azure Durable Functions Knowledge](https://docs.microsoft.com/en-us/azure/azure-functions/durable-functions-overview)

## 1. Create a .NET Core lib project

Create a .NET Core lib project.

``` powershell
dotnet new lib --name durable
cd dni
rm .\Class1.cs
```

## 2. Add the required references to be able to use Azure Functions API and the Durable Task Extensions

Be sure to add the following packages to the project. Sample code will not work with the latest versions of some of the Microsoft.AspNetCore.* packages, so pay special attention to the version parameter.

**Note:**

* **Microsoft.AspNetCore.Mvc.WebApiCompatShim** is needed to use the **ReadAsAsync** extension (Assembly: System.Net.Http.Formatting) in one of the functions.
* **Microsoft.Azure.WebJobs.Script.ExtensionsMetadataGenerator** is need to generate, during build time, the extension file needed to run durable fucntions.

``` powershell
dotnet add package Microsoft.AspNetCore.Http -v 2.0.0
dotnet add package Microsoft.AspNetCore.Mvc -v 2.0.0
dotnet add package Microsoft.AspNetCore.Mvc.WebApiCompatShim -v 2.0.0
dotnet add package Microsoft.Azure.WebJobs -v 3.0.0-beta4
dotnet add package Microsoft.Azure.WebJobs.Extensions.DurableTask -v 1.0.0-beta
dotnet add package Microsoft.Azure.WebJobs.Script.ExtensionsMetadataGenerator -v 1.0.0-beta2
```

## 3. Create the wwwroot and the folders needed to hold the functions

Create some foders. The first one **wwwroot** will hold all the functions and global configuration. The others will hold the specific files for each of the 3 azure functions you'll deploy:

``` powershell
md wwwroot
md wwwroot/httpstart
md wwwroot/activity
md wwwroot/orchestrator
```

## 4. Create host.json file in the wwwroot folder

Create a **host.json** file in the **wwwroot** folder with the following contents:

``` json
{ }
```

## 5. Create the activity function

In the **wwwroot/activity** folder create a **functions.json** file with the following contents:

``` json
{
    "bindings": [
        {
            "name": "name",
            "type": "activityTrigger",
            "direction": "in"
        }
    ],
    "disabled": false,
    "scriptFile": "..//bin//durable.dll",
    "entryPoint": "DurableFunctions.Activity.Run"
}
```

Now create a **Activity.cs** file with the following contents:

``` csharp
using System.Collections.Generic;
using System.Threading.Tasks;
using Microsoft.Azure.WebJobs;

namespace DurableFunctions
{
    public class Activity
    {
        public static string Run(string name)
        {
            return $"Hello {name}!";
        }
    }
}
```

**Note**: this function uses an [Activity Trigger](https://docs.microsoft.com/en-us/azure/azure-functions/durable-functions-bindings).

## 6. Create an orchestrator function

In the **wwwroot/orchestrator** folder create a **functions.json** file with the following contents:

``` json
{
    "bindings": [
        {
            "name": "context",
            "type": "orchestrationTrigger",
            "direction": "in"
        }
    ],
    "disabled": false,
    "scriptFile": "..//bin//durable.dll",
    "entryPoint": "DurableFunctions.Orchestrator.Run"
}
```

Now create a **Orchestrator.cs** file with the following contents:

``` csharp
using System.Collections.Generic;
using System.Threading.Tasks;
using Microsoft.Azure.WebJobs;

namespace DurableFunctions
{
    public class Orchestrator
    {

        public static async Task<List<string>> Run(
            DurableOrchestrationContext context)
        {
            var outputs = new List<string>();

            outputs.Add(await context.CallActivityAsync<string>("activity", "Tokyo"));
            outputs.Add(await context.CallActivityAsync<string>("activity", "Seattle"));
            outputs.Add(await context.CallActivityAsync<string>("activity", "London"));

            // returns ["Hello Tokyo!", "Hello Seattle!", "Hello London!"]
            return outputs;
        }
    }
}
```

**Note**: this function uses an [Orchestration Trigger](https://docs.microsoft.com/en-us/azure/azure-functions/durable-functions-bindings).

## 7. Create the Orchestrator Client

In the **wwwroot/httpstart** folder create a **functions.json** file with the following contents:

``` json
{
    "bindings": [
        {
            "authLevel": "anonymous",
            "name": "req",
            "type": "httpTrigger",
            "direction": "in",
            "route": "orchestrators/{functionName}",
            "methods": [
                "post"
            ]
        },
        {
            "name": "starter",
            "type": "orchestrationClient",
            "direction": "in"
        }
    ],
    "disabled": false,
    "scriptFile": "..//bin//durable.dll",
    "entryPoint": "DurableFunctions.HttpStart.Run"
}
```

Now create a **HttpStart.cs** file with the following contents:

``` csharp
using System;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Threading.Tasks;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Host;

namespace DurableFunctions
{
    public class HttpStart
    {
        public static async Task<HttpResponseMessage> Run(
            HttpRequestMessage req,
            DurableOrchestrationClient starter,
            string functionName,
            TraceWriter log)
        {
            // Function input comes from the request content.
            dynamic eventData = await req.Content.ReadAsAsync<object>();
            string instanceId = await starter.StartNewAsync(functionName, eventData);

            log.Info($"Started orchestration with ID = '{instanceId}'.");

            var res = starter.CreateCheckStatusResponse(req, instanceId);
            res.Headers.RetryAfter = new RetryConditionHeaderValue(TimeSpan.FromSeconds(10));
            return res;
        }
    }
}
```

**Note**: this function uses an http trigger and an [Orchestration client](https://docs.microsoft.com/en-us/azure/azure-functions/durable-functions-bindings).

## 8. Create a dockerfile

Create a **dockerfile** with the following contents:

``` dockerfile
FROM microsoft/azure-functions-runtime:2.0.0-jessie

# If we don't set WEBSITE_HOSTNAME Azure Function triggers other than HttpTrigger won't run. (FunctionInvocationException: Webhooks are not configured)
# I've created a pull request to add this variable to the original dockerfile: https://github.com/Azure/azure-webjobs-sdk-script/pull/2285
ENV WEBSITE_HOSTNAME=localhost:80

# Copy all Function files and binaries to /home/site/wwwroot
ADD wwwroot /home/site/wwwroot

# Functions must live in: /home/site/wwwroot
# We expect to receive the Storage Account Connection String from the command line.
ARG STORAGE_ACCOUNT

# Set the AzureWebJobsStorage Environment Variable. Otherwise Durable Functions Extensions won't work.
ENV AzureWebJobsStorage=$STORAGE_ACCOUNT
```

Note that once you execute a docker build on the file all the contents of the **wwwroot** folder will be copied to the **/home/site/wwwroot** inside the image.

## 9. Build the project and the docker image

Build the .NET Core project so you get the precompiled funtions and then build the docker image:

Note: you'll need a valid Azure Storage Account connection string.

``` powershell
dotnet build -o .\wwwroot\bin\
docker build --build-arg STORAGE_ACCOUNT="[Valid Azure Storage Account Connection String]" -t durable .
```

## 10. Create a container and test the function

Run the following to run the Azure Function in a container:

``` powershell
docker run -p 5000:80 durable
```

Now POST to the orchestrator client to start the durable functions:

``` bash
curl -X POST http://localhost:5000/api/orchestrators/orchestrator -i -H "Content-Length: 0"
```

You can download all code and files [here](https://github.com/cmendible/dotnetcore.samples/tree/master/azure.durable.function.docker).

**Note**: The function code is based on the original [Azure Functions Samples](https://github.com/Azure/azure-functions-durable-extension).

Happy coding!
