---
author: Carlos Mendible
categories:
- dotnet
date: "2017-07-04T12:59:05Z"
description: Deploy your ASP.NET Core Web API to AWS Lambda
images: ["/wp-content/uploads/2017/07/AWS-Lambda-DotNet-Core.png"]
tags: ["aspNetCore", "AWS", "Lambda", "Serverless", "WebAPI"]
title: Deploy your ASP.NET Core Web API to AWS Lambda
url: /2017/07/04/deploy-your-asp-net-core-web-api-to-aws-lambda/
---
Today I'll show you how to **Deploy your ASP.NET Core Web API to AWS Lambda**.

First be aware of the following prerequisites:

  * You'll need an AWS account: <a href="https://aws.amazon.com/" target="_blank">https://aws.amazon.com/</a>
  * You'll need AWS CLI installed: <a href="http://docs.aws.amazon.com/cli/latest/userguide/installing.html" target="_blank">http://docs.aws.amazon.com/cli/latest/userguide/installing.html</a>
  * You'll need your AWS security credentials
  * Some basic knowledge on <a href="http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacks.html" target="_blank">AWS Stacks</a>, <a href="https://aws.amazon.com/es/api-gateway/" target="_blank">API Gateway</a>, <a href="https://aws.amazon.com/es/lambda/" target="_blank">Lambda</a> and <a href="http://docs.aws.amazon.com/AmazonS3/latest/dev/UsingBucket.html" target="_blank">Buckets</a>

Now let's start:

## 1. Create a folder for your new project
---
Open a command promt an run 
    
``` powershell
mkdir aws.lambda
```
## 2. Create the project
---

``` powershell
cd aws.lambda
dotnet new webapi
```

## 3. Replace the contents of aws.lambda.csproj
---
You need to target **netcoreapp1.0** in order for your Web API to work on AWS Lambda. You'll also need to reference some nuget packages from aws so replace the contents of the **aws.lamda.csproj** file with the following lines:

    
``` xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>netcoreapp1.0</TargetFramework>
    <UserSecretsId>aspnet-aws.lambda-FD62E418-7AF9-462F-B391-DD806F38F03A</UserSecretsId>
    <PreserveCompilationContext>false</PreserveCompilationContext>
  </PropertyGroup>

  <ItemGroup>
    <Folder Include="wwwroot\" />
  </ItemGroup>

<ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.Server.IISIntegration" Version="1.0.0" />
    <PackageReference Include="Microsoft.AspNetCore.Server.Kestrel" Version="1.0.1" />
    <PackageReference Include="Microsoft.AspNetCore.Mvc" Version="1.0.1" />
    <PackageReference Include="Microsoft.AspNetCore.Routing" Version="1.0.1" />
    <PackageReference Include="Microsoft.Extensions.Configuration.EnvironmentVariables" Version="1.0.0" />
    <PackageReference Include="Microsoft.Extensions.Configuration.FileExtensions" Version="1.0.0" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="1.0.0" />
    <PackageReference Include="Microsoft.Extensions.Logging" Version="1.0.0" />
    <PackageReference Include="Microsoft.Extensions.Options.ConfigurationExtensions" Version="1.0.0" />

    <PackageReference Include="AWSSDK.Extensions.NETCore.Setup" Version="3.3.0.3" />

    <PackageReference Include="Amazon.Lambda.Core" Version="1.0.0" />
    <PackageReference Include="Amazon.Lambda.Serialization.Json" Version="1.1.0" />
    <PackageReference Include="Amazon.Lambda.AspNetCoreServer" Version="0.10.1-preview1" />
    <PackageReference Include="Amazon.Lambda.Logging.AspNetCore" Version="1.1.0" />
  </ItemGroup>
  
  <ItemGroup>
    <DotNetCliToolReference Include="Microsoft.VisualStudio.Web.CodeGeneration.Tools" Version="1.0" />
    <DotNetCliToolReference Include="Amazon.Lambda.Tools" Version="1.6.0" />
  </ItemGroup>

</Project>
```


## 4. Restore the packages
---
You just changed the project files so restore the packages:
    
``` powershell
dotnet restore
```
## 5. Rename Program.cs and replace it's contents
---
Rename **Program.cs** to **LocalEntrypoint.cs** and replace the contents of the file with the following lines:
    
``` csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Builder;

namespace aws.lambda
{
    /// <summary>
    /// The Main function can be used to run the ASP.NET Core application locally using the Kestrel webserver.
    /// </summary>
    public class LocalEntryPoint
    {
        public static void Main(string[] args)
        {
            var host = new WebHostBuilder()
                .UseKestrel()
                .UseContentRoot(Directory.GetCurrentDirectory())
                .UseIISIntegration()
                .UseStartup<Startup>()
                .Build();

            host.Run();
        }
    }
}
```
    
Note: This will be the entry point while you are developing.

## 6. Create a file LambdaFunction.cs
---
Create a **LambdaFunction.cs** file and copy the following code 
    
``` csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Net;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Hosting;
using System.IO;

namespace aws.lambda
{
    public class LambdaFunction : Amazon.Lambda.AspNetCoreServer.APIGatewayProxyFunction
    {
        protected override void Init(IWebHostBuilder builder)
        {
            builder
                .UseContentRoot(Directory.GetCurrentDirectory())
                .UseStartup<Startup>()
                .UseApiGateway();
        }
    }
}
```
    
Note: This class will be the entry point once you deploy the Web API to AWS Lambda.

## 7. Replace the contents of Startup.cs
---
Replace the contents of the **Startup.cs** file with the following code 
          
``` csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;

namespace aws.lambda
{
    public class Startup
    {
        public Startup(IHostingEnvironment env)
        {
            var builder = new ConfigurationBuilder()
                .SetBasePath(env.ContentRootPath)
                .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
                .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true);

            builder.AddEnvironmentVariables();
            Configuration = builder.Build();
        }

        public static IConfigurationRoot Configuration { get; private set; }

        // This method gets called by the runtime. Use this method to add services to the container
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddMvc();

            // Pull in any SDK configuration from Configuration object
            services.AddDefaultAWSOptions(Configuration.GetAWSOptions());
        }

        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline
        public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
        {
            loggerFactory.AddLambdaLogger(Configuration.GetLambdaLoggerOptions());
            app.UseMvc();
        }
    }
}
```
          
Note: Check lines 34 & 40 for special AWS configurations.

## 8. Create aws-lambda-tools-defaults.json
Create a aws-lambda-tools-defaults.json file with the following code 
          
``` json
{
    "Information": [
        "This file provides default values for the deployment wizard inside Visual Studio and the AWS Lambda commands added to the .NET Core CLI.",
        "To learn more about the Lambda commands with the .NET Core CLI execute the following command at the command line in the project root directory.",
        "dotnet lambda help",
        "All the command line options for the Lambda command can be specified in this file."
    ],
    "profile": "default",
    "region": "eu-west-1",
    "configuration": "Release",
    "framework": "netcoreapp1.0",
    "function-runtime": "dotnetcore1.0",
    "template": "serverless.template"
}
```
          
Note: Be aware of teh region specified in this file.
      
## 9. Create serverless.template
---      
Create a file: serverless.template with the following contents 
          
``` json
{
  "AWSTemplateFormatVersion" : "2010-09-09",
  "Transform" : "AWS::Serverless-2016-10-31",
  "Description" : "Starting template for an AWS Serverless Application.",
  "Parameters" : {
  },
  "Resources" : {
    "DefaultFunction" : {
      "Type" : "AWS::Serverless::Function",
      "Properties": {
        "Handler": "aws.lambda::aws.lambda.LambdaFunction::FunctionHandlerAsync",
        "Runtime": "dotnetcore1.0",
        "CodeUri": "",
        "Description": "Default function",
        "MemorySize": 256,
        "Timeout": 30,
        "Role": null,
        "Policies": [ "AWSLambdaFullAccess" ],
        "Events": {
          "PutResource": {
            "Type": "Api",
            "Properties": {
              "Path": "/{proxy+}",
              "Method": "ANY"
            }
          }
        }
      }
    }
  },
  "Outputs" : {
  }
}
```
          
Note: be aware of the Handler property in this file. It should be in the form:<br /> **[Assembly Name]::[Namespace].LambdaFunction::FunctionHandlerAsync**
      
## 10. Build the application
--- 
Build the application:
      
          
``` powershell
dotnet build
```
      
##  11. Deploy to AWS Lambda
---        
Run the following commands (you'll need your access key for AWS):

``` powershell
aws configure
aws s3api create-bucket --bucket mylambdabucket --region eu-west-1 --create-bucket-configuration LocationConstraint=eu-west-1 
dotnet lambda deploy-serverless
```

## 12. Get your api id
---      
Run the following commands and copy the api id for your stack:
      
          
``` powershell
aws apigateway get-rest-apis
```
         
## 13. Browse to your web api
---      
You are good to go so browse to:<br /> **https://[api id from step 11].execute-api.eu-west-1.amazonaws.com/Prod/api/values**

The browser should show: [&#8220;value1&#8243;,"value2"]  

You can get the code here: <a href="https://github.com/cmendible/dotnetcore.samples/tree/main/aws.lambda">https://github.com/cmendible/dotnetcore.samples/tree/main/aws.lambda</a>

Hope it helps!        