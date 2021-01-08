---
author: Carlos Mendible
categories:
- dotnet
crosspost_to_medium: true
date: "2021-01-08T10:00:00Z"
description: 'ASP.NET Core OpenTelemetry Logging'
images: ["/assets/img/posts/dapr.png"]
published: true
tags: ["opentelemetry"]
title: 'ASP.NET Core OpenTelemetry Logging'
---

As you may know I've been collaborating with [Dapr](https://dapr.io/) and I've learned that one of the things it enables you to do is to collect traces with the use of the [OpenTelemetry Collector](https://github.com/open-telemetry/opentelemetry-collector) and push the events to [Azure Application Insights](https://docs.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview?WT.mc_id=AZ-MVP-5002618).

After some reading I went and check if I could also write my ASP.NET Core applications to log using the OpenTelemetry Log and Event record definition:

Field Name     |Description
---------------|--------------------------------------------
Timestamp      |Time when the event occurred.
TraceId        |Request trace id.
SpanId         |Request span id.
TraceFlags     |W3C trace flag.
SeverityText   |The severity text (also known as log level).
SeverityNumber |Numerical value of the severity.
Name           |Short event identifier.
Body           |The body of the log record.
Resource       |Describes the source of the log.
Attributes     |Additional information about the event.

So here is how you can do it:

## Create an ASP.NET Core application

``` shell
dotnet new webapi -n aspnet.opentelemetry
cd aspnet.opentelemetry
```

## Add a reference to the OpenTelemtry libraries

``` shell
dotnet add package OpenTelemetry.Extensions.Hosting --prerelease 
dotnet add package OpenTelemetry.Exporter.Console --prerelease   
```

## Modify the CreateHostBuilder method in Program.cs

``` csharp
public static IHostBuilder CreateHostBuilder(string[] args) =>
  Host.CreateDefaultBuilder(args)
      .ConfigureLogging(logging =>
      {
          logging.ClearProviders();
          logging.AddOpenTelemetry(options =>
          {
              options.AddProcessor(new SimpleExportProcessor<LogRecord>(new ConsoleExporter<LogRecord>(new ConsoleExporterOptions())));
          });
      })
      .ConfigureWebHostDefaults(webBuilder =>
      {
          webBuilder.UseStartup<Startup>();
      });
```

The code first clears all logging providers and then adds OpenTelemetry using the `SimpleExportProcessor` and `ConsoleExporter`. 

Please check the [OpenTelemetry .NET API](https://github.com/open-telemetry/opentelemetry-dotnet) repo to learn more.

## Modify the log settings

Edit the `appsettings.Development.json` in order to configure the default log settings using the OpenTelemetry provider:

``` json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*"
}
```

## Test the program

Run the following command to test the application.

``` shell
dotnet run
```

You should see the logs, formatted as OpenTelemtry log records, written in the console:

[]()                          | []()
------------------------------|---------------------------------
LogRecord.TraceId:            | 00000000000000000000000000000000
LogRecord.SpanId:             | 0000000000000000
LogRecord.Timestamp:          | 2021-01-08T10:36:26.1338054Z
LogRecord.EventId:            | 0
LogRecord.CategoryName:       | Microsoft.Hosting.Lifetime
LogRecord.LogLevel:           | Information
LogRecord.TraceFlags:         | None
LogRecord.State:              | Application is shutting down...

Both `TraceId` and `SpanId` will get filled when a request is handled by your application and if the call is made by Dapr it will respect the values sent by it so you can correlate traces and logs improving the observability of your solution.

Hope it helps! and please find the complete code [here](https://github.com/cmendible/dotnetcore.samples/tree/main/aspnet.opentelemetry)