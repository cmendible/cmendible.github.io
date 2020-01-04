---
author: Carlos Mendible
categories:
- DevOps
- dotNetCore
date: "2016-12-11T12:49:54Z"
description: Debug ASP.NET Core on Docker with Visual Studio Code
image: /wp-content/uploads/2016/09/dotnetdocker.jpg
# tags: aspNetCore Debug Docker VisualStudioCode
title: Debug ASP.NET Core on Docker with Visual Studio Code
---
Last Thursday I started reading the Free eBook: <a href="https://aka.ms/dockerlifecycleebook" target="_blank">"Containerized Docker Application Lifecycle with Microsoft Tools and Platform"</a> by Cesar de la Torre. The book is really easy to read and really gives the reader a glimpse on the way to approach the overall application lifecycle when using containers and Microsoft Technologies. One of the interesting things he mentions is the ability to debug your source code inside a container using Visual Studio or Visual Studio Code, so I decided to try it out and **Debug ASP.NET Core on Docker with Visual Studio Code**

First be aware of the following prerequisites:

| **Prerequisites** |  **Command** |
|---|---|
.NET Core 1.1 SDK for Windows | Download and install: <a href="https://go.microsoft.com/fwlink/?LinkID=835014" target="_blank">.NET Core 1.1 SDK for Windows</a>|
|Docker for Windows| Download and install: <a href="https://download.docker.com/win/stable/InstallDocker.msi" target="_blank">Docker for Windows (stable)</a>|
|Visual Studio Code and C# Plugin | Download and install from here: <a href="https://code.visualstudio.com/" target="_blank">Code</a>|
|Node.js| Download and install from here: <a href="https://nodejs.org/dist/v6.9.2/node-v6.9.2-x64.msi" target="_blank">Node.js v6.9.2</a>|
|Yeoman| From a Command Prompt run the following: **npm install -g yo**|
|Generator-docker|Run the following command: **npm install -g generator-docker**|
|Bower| Run the following command: **npm install -g bower**|
|Gulp| Run the following command: **npm install -g gulp**|

Now let's create an ASP .NET Core application and debug it inside Docker:

## 1. Create an ASP .NET Core application
---
Open a command prompt and run 
          
``` powershell
    md debug.on.docker
    cd debug.on.docker
    dotnet new -t web
    dotnet restore
    code .
```

## 2. Change your application port in Program.cs
---
Let's use port **5000** to host the application
          
``` csharp
   public static void Main(string[] args)
   {
            var host = new WebHostBuilder()
                .UseUrls("http://*:5000")
                .UseKestrel()
                .UseContentRoot(Directory.GetCurrentDirectory())
                .UseIISIntegration()
                .UseStartup<Startup>()
                .Build();

            host.Run();
   }
```  
      
## 3. Create the artifacts needed to Debug on Docker
---
Run the following command in the root folder of your project.
          
``` powershell
    yo docker
```

Answer the questions as follows:

``` powershell
What language is your project using? .NET Core
Which version of .NET Core is your project using? rtm
Does your project use a web server? Yes
Which port is your app listening to? 5000
What do you want to name your image? debug.on.docker
What do you want to name your service? debug.on.docker
What do you want to name your compose project? debugondocker
```   
      
## 4. Change dockerTask.ps1
---      
The Yeoman generator is not ready for .Net Core 1.1 so change **dockerTask.ps1** line **47** with 
          
``` powershell
   $framework = "netcoreapp1.1"
```
      
## 5. Change the docker files
---
Change the fist line of both **Dockerfile** and **Dockerfile.debug** in order to use the latest image for .NET Core 1.1
          
``` powershell
   FROM microsoft/dotnet
```

## 6. Debug your application
---
Place a break point in your code (i.e. line 14 of Program.cs) and hit **F5** in Visual Studio Code. 

The first time it will take some time, but your application will run as a container and you'll be able to debug it!!!
            
You can get a copy of the code here: <a href="https://github.com/cmendible/dotnetcore.samples/tree/master/debug.on.docker">https://github.com/cmendible/dotnetcore.samples/tree/master/debug.on.docker</a>
              
Hope it helps!