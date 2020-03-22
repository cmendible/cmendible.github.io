---
author: Carlos Mendible
categories:
- azure
- dotnet
crosspost_to_medium: true
date: "2017-12-28T15:13:00Z"
description: Run a Precompiled .NET Core Azure Function in a Container
image: /assets/img/posts/azurefunctions.jpg
published: true
tags: ["Docker", "AzureFunctions", "Dockerfile", "Serverless"]
title: Run a Precompiled .NET Core Azure Function in a Container
url: /2017/12/28/run-a-precomplied-net-core-azure-function-in-a-container/
---

So this morning I found my self browsing through the images Microsoft has published in the [Docker Hub](https://hub.docker.com/u/microsoft/) and then I saw this one: [microsoft/azure-functions-runtime](https://hub.docker.com/r/microsoft/azure-functions-runtime/) and decided to **Run a Precompiled .NET Core Azure Function in a Container**.

**Prerequisites**:

* [Docker](https://www.docker.com) installed and basic knowledge.
* [.NET Core](https://www.microsoft.com/net/download/)

## 1. Create a .NET Core lib project

Create a .NET Core lib project.

``` powershell
dotnet new lib --name dni
cd dni
rm .\Class1.cs
```

## 2. Add the required references to be able to use Azure Functions API

Be sure to add the following packages to the project. I couldn't make the sample work with the latest versions of some of the Microsoft.AspNetCore.* packages, so pay special attention to the version parameter.

``` powershell
dotnet add package Microsoft.AspNetCore.Http -v 2.0.0
dotnet add package Microsoft.AspNetCore.Mvc -v 2.0.0
dotnet add package Microsoft.Azure.WebJobs -v 3.0.0-beta4
dotnet add package Microsoft.Extensions.Primitives -v 2.0.0
dotnet add package Newtonsoft.Json -v 10.0.3
```

## 3. Modify the Output Path of the project

Edit dni.csproj and the OutputPath property with the following value: **wwwroot\bin**. The project contents should now look like the following text:

``` xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <OutputPath>wwwroot\bin</OutputPath>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.Http" Version="2.0.0" />
    <PackageReference Include="Microsoft.AspNetCore.Mvc" Version="2.0.0" />
    <PackageReference Include="Microsoft.Azure.WebJobs" Version="3.0.0-beta4" />
    <PackageReference Include="Microsoft.Extensions.Primitives" Version="2.0.0" />
    <PackageReference Include="Newtonsoft.Json" Version="10.0.3" />
  </ItemGroup>
</Project>
```

## 4. Create the wwwroot and validate folders

Create two folders. The first one **wwwroot** will hold all the functions and global configuration. The second one: **validate** will hold the configuration of the **validate** function we'll be deploying.

``` powershell
md wwwroot
md wwwroot/validate
```

## 5. Create host.json file in the wwwroot folder

Create a **host.json** file in the **wwwroot** folder with the following contents:

``` json
{ }
```

## 6. Create function.json and run.csx files

In the **wwwroot/validate** folder create a **functions.json** file with the following contents:

``` json
{
  "bindings": [
    {
      "authLevel": "anonymous",
      "name": "req",
      "type": "httpTrigger",
      "direction": "in"
    }
  ],
  "scriptFile": "..//bin//netstandard2.0//dni.dll",
  "entryPoint": "Dni.HttpTrigger.Run",
  "disabled": false
}
```

Note that in **functions.json** we are specifying the function trigger **type** as **httpTrigger** and also specifying the **scriptFile** and **entryPoint**.

Now create a **run.csx** file with the following contents:

``` csharp
//leave this file empty - all the logic is placed within our precompiled function
```

## 7. Create an HttpTrigger.cs file to hold your function

Create a **HttpTrigger.cs** file with the following contents:

``` csharp
namespace Dni
{
    using System.Collections.Generic;
    using System.Linq;
    using Microsoft.AspNetCore.Http;
    using Microsoft.AspNetCore.Mvc;
    using Microsoft.Azure.WebJobs.Host;

    public class HttpTrigger
    {
        public static IActionResult Run(HttpRequest req, TraceWriter log)
        {
            log.Info("DNI Validation function is processing a request.");

            string dni = req.Query["dni"];

            return dni != null
                ? (ActionResult)new OkObjectResult(ValidateDNI(dni))
                : new BadRequestObjectResult("Please pass a dni on the query string");
        }

        public static bool ValidateDNI(string dni)
        {
            var table = "TRWAGMYFPDXBNJZSQVHLCKE";
            var foreignerDigits = new Dictionary<char, char>()
        {
            { 'X', '0'},
            { 'Y', '1'},
            { 'Z', '2'}
        };
            var numbers = "1234567890";
            var parsedDNI = dni.ToUpper();
            if (parsedDNI.Length == 9)
            {
                var checkDigit = parsedDNI[8];
                parsedDNI = parsedDNI.Remove(8);
                if (foreignerDigits.ContainsKey(parsedDNI[0]))
                {
                    parsedDNI = parsedDNI.Replace(parsedDNI[0], foreignerDigits[parsedDNI[0]]);
                }

                return parsedDNI.Length == parsedDNI.Where(n => numbers.Contains(n)).Count() &&
                    table[int.Parse(parsedDNI) % 23] == checkDigit;
            }

            return false;
        }
    }
}
```

Cause I didn't want to create a simple Hello World sample, this Azure Function Code takes the **dni** parameter from the query string and validates if the given value is a valid **Spanish National Identity Document** number.

## 8. Create a dockerfile

Create a **dockerfile** with the following contents:

``` dockerfile
FROM microsoft/azure-functions-runtime:v2.0.0-beta1
ENV AzureWebJobsScriptRoot=/home/site/wwwroot
ADD wwwroot /home/site/wwwroot
```

Note that once you execute a docker build on the file all the contents of the **wwwroot** folder will be copied to the **/home/site/wwwroot** inside the image.

## 9. Build the project and the docker image

Build the .NET Core project so you get a precompiled funtion and then build the docker image:

``` powershell
dotnet build
docker build -t dni .
```

## 10. Create a container and test the function

Run the following to run the Azure Function in a container:

``` powershell
docker run -p 5000:80 dni
```

Now navigate to: [http://localhost:5000/api/validate?dni=54495436H](http://localhost:5000/api/validate?dni=54495436H) and the page should return **true**.

You can download all code and files [here](https://github.com/cmendible/dotnetcore.samples/tree/master/azure.function.docker).

Hope it helps!