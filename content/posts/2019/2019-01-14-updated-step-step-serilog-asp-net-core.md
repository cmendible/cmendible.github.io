---
author: Carlos Mendible
categories:
- dotnet
crosspost_to_medium: true
date: "2019-01-14T18:00:00Z"
description: 'Updated Step by step: Serilog with ASP.NET Core'
image: /wp-content/uploads/2016/09/serilog.png
published: true
tags: ["aspNetCore", "SeriLog"]
title: 'Updated Step by step: Serilog with ASP.NET Core'
url: /2019/01/14/updated-step-step-serilog-asp-net-core/
---

Many of you come to my site to read the post [Step by step: Serilog with ASP.NET Core](https://carlos.mendible.com/2016/09/19/step-step-serilog-asp-net-core/) which I wrote in 2016 and is completely out of date, so with this post I will show you how to setup [Serilog](https://serilog.net/) to work with your ASP.NET Core 2.2 applications.

## 1. Create an ASP.NET Core project
---

``` bash
    md aspnet.serilog.sample
    cd aspnet.serilog.sample
    dotnet new mvc
```

## 2. Add the following dependencies to your project
---

``` bash
    dotnet add package Serilog.AspNetCore
    dotnet add package Serilog.Extensions.Logging
    dotnet add package Serilog.Sinks.ColoredConsole
```

## 3. Change your Program.cs file to look like the following
---

``` csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using Serilog;
using Serilog.Core;
using Serilog.Events;

namespace aspnet.serilog.sample
{
    public class Program
    {
        public static void Main(string[] args)
        {
            Log.Logger = new LoggerConfiguration()
               .Enrich.FromLogContext()
               .MinimumLevel.Debug()
               .WriteTo.ColoredConsole(
                   LogEventLevel.Verbose,
                   "{NewLine}{Timestamp:HH:mm:ss} [{Level}] ({CorrelationToken}) {Message}{NewLine}{Exception}")
                   .CreateLogger();

            try
            {
                CreateWebHostBuilder(args).Build().Run();
            }
            finally
            {
                Log.CloseAndFlush();
            }
        }

        public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
            WebHost.CreateDefaultBuilder(args)
                .UseSerilog()
                .UseStartup<Startup>();
    }
}
```

## 4. Inject the logger to your services or controllers
---

Change the home controller and log some actions:

``` csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using aspnet.serilog.sample.Models;
using Microsoft.Extensions.Logging;

namespace aspnet.serilog.sample.Controllers
{
    public class HomeController : Controller
    {
        ILogger<HomeController> logger;

        public HomeController(ILogger<HomeController> logger)
        {
            this.logger = logger;
        }

        public IActionResult Index()
        {
            this.logger.LogDebug("Index was called");
            return View();
        }

        public IActionResult Privacy()
        {
            return View();
        }

        [ResponseCache(Duration = 0, Location = ResponseCacheLocation.None, NoStore = true)]
        public IActionResult Error()
        {
            return View(new ErrorViewModel { RequestId = Activity.Current?.Id ?? HttpContext.TraceIdentifier });
        }
    }
}
```

Download the complete code [here](https://github.com/cmendible/dotnetcore.samples/tree/master/aspnet.serilog.sample2.2)

Hope it helps!