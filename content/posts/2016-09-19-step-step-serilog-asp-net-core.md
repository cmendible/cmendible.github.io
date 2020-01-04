---
author: Carlos Mendible
categories:
- dotNetCore
date: "2016-09-19T09:32:57Z"
description: 'Step by step: Serilog with ASP.NET Core'
image: /wp-content/uploads/2016/09/serilog.png
# tags: aspNetCore SeriLog
title: 'Step by step: Serilog with ASP.NET Core'
url: /2016/09/19/step-step-serilog-asp-net-core/
---

## **NOTE**: This post is out of date. Please read: [Updated Step by step: Serilog with ASP.NET Core](https://carlos.mendible.com/2019/01/14/updated-step-step-serilog-asp-net-core/) for an update. 

Last week I wrote about <a href="https://carlos.mendible.com/2016/09/11/netcore-and-microsoft-bot-framework/" target="_blank">.NET Core and Microsoft Bot Framework</a> and I'm still learning and playing with it. The thing is that once I implemented more features and deployed the bot to Azure it didn't work. So I had to find a way to log and trace what was happening in order to diagnose and fix the problem.

This time I decided to give a chance to <a href="https://serilog.net/" target="_blank">Serilog</a> and as you should know by now, getting the correct dependencies to work with .Net Core is not always easy, so here is my **Step by step: Serilog with ASP.NET Core**

## 1. Add the following dependencies in your project.json file

``` json
    "Serilog": "2.2.0",
    "Serilog.Extensions.Logging": "1.2.0",
    "Serilog.Sinks.RollingFile": "2.0.0",
    "Serilog.Sinks.File": "3.0.0"
```

## 2. Add the following lines to the constructor of your Startup class
---  

``` csharp
     Log.Logger = new LoggerConfiguration()
        .MinimumLevel.Debug()
        .WriteTo.RollingFile(Path.Combine(env.ContentRootPath, "log-{Date}.txt"))
        .CreateLogger();
```

## 3. Add the following line to the configure method of your Startup class
---  

``` csharp
     loggerFactory.AddSerilog();
```

## 4. Inject the logger to your services or controllers
---

``` csharp
    public class Chat : IChat
    {
        // Instancia del logger
        ILogger logger;

        // Injectamos el logger en el constructor
        public Chat(ILogger logger)
        {
            this.logger = logger;
        }

        // Método de envÃ­o de los mensajes
        public virtual void SendMessage(string message)
        {
            try
            {
                // Enví­o de Mensaje
            }
            catch (System.Exception ex)
            {
                // Enviamos al log los errores.
                this.logger.LogError(ex.Message);
            }
        }
    }
```

Hope it helps!