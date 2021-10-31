---
author: Carlos Mendible
categories:
- dotnet
date: "2017-01-16T14:54:55Z"
description: '.Net Core and NancyFX: can writing a WebApi get any simpler?'
images: ["/wp-content/uploads/2017/01/nancy.jpg"]
tags: ["aspnetcore", "nancyfx"]
title: '.Net Core and NancyFX: can writing a WebApi get any simpler?'
url: /2017/01/16/net-core-and-nancyfx-can-writing-a-webapi-get-any-simpler/
---
Last Thursday I attended a Meetup hosted by my friends of <a href="https://twitter.com/mscodersmadrid" target="_blank">@MsCodersMadrid</a> in Madrid where, thanks to <a href="https://twitter.com/snavarropino" target="_blank">@snavarropino</a>, I learned a bit about the **<a href="http://nancyfx.org/" target="_blank">NancyFX</a>** open source framework. 

I really couldn't believe my eyes when I saw how simple it is to use **<a href="http://nancyfx.org/" target="_blank">NancyFX</a>** to write a Web API. Two of the things that got my attention were: the out of the box content negotiation and zero configuration dependency injection. 

What are we waiting for? Lets's put in the mix **.Net Core and <a href="http://nancyfx.org/" target="_blank">NancyFX</a>** and ask ourselves: **can a WebApi get any simpler?**.

Steps:

## 1. Create the application
---
Open a command prompt and run 
    
``` powershell
    md nancyfx.sample
    cd nancyfx.sample
    dotnet new
    dotnet restore
    code .
```

## 2. Replace the contents of project.json
---
Replace the contents on **project.json** file. Be sure to include the references to **Microsoft.AspNetCore.Owin** and **Nancy**
    
``` json
{
  "dependencies": {
    "Microsoft.NETCore.App": {
      "version": "1.1.0-preview1-001153-00",
      "type": "platform"
    },
    "Microsoft.AspNetCore.Server.IISIntegration": "1.0.0",
    "Microsoft.AspNetCore.Server.Kestrel": "1.0.1",
    "Microsoft.AspNetCore.Owin": "1.0.0",
    "Nancy": "2.0.0-clinteastwood"
  },
  "tools": {},
  "frameworks": {
    "netcoreapp1.1": {
      "imports": [
        "dotnet5.6",
        "dnxcore50",
        "portable-net45+win8"
      ]
    }
  },
  "buildOptions": {
    "debugType": "portable",
    "emitEntryPoint": true,
    "preserveCompilationContext": true
  },
  "runtimeOptions": {
    "configProperties": {
      "System.GC.Server": true
    }
  },
  "publishOptions": {},
  "scripts": {},
  "tooling": {
    "defaultNamespace": "WebApplication"
  }
}
```

## 3. Replace the contents of Program.cs
---
Let's host the Web API with AspNetCore 
    
``` csharp
using System.IO;
using Microsoft.AspNetCore.Hosting;

namespace WebApplication
{
    public class Program
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

## 4. Add a Startup.cs
---
This one-liner hooks **<a href="http://nancyfx.org/" target="_blank">NancyFX</a>** to our application. 
    
``` csharp
using Microsoft.AspNetCore.Builder;
using Nancy.Owin;

namespace WebApplication
{
    public class Startup
    {
        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app)
        {
            app.UseOwin(x => x.UseNancy());
        }
    }
}
```

## 5. Create a service to return some random message
---
Add a new file (i.e BaconIpsumService.cs) to hold the application logic and add the following sample content 
    
``` csharp
using System.Net.Http;
using System.Threading.Tasks;

namespace WebApplication
{
    public class BaconIpsumMessage
    {
        public string Body {get; set;}
    }

    public interface IBaconIpsumService
    {
        Task<BaconIpsumMessage> GenerateAsync();
    }

    public class BaconIpsumService : IBaconIpsumService
    {
        public async Task<BaconIpsumMessage> GenerateAsync()
        {
            using (var client = new HttpClient())
            {
                var message = new BaconIpsumMessage();
                // Post the message
                message.Body =  await client.GetStringAsync(
                    $"https://baconipsum.com/api/?type=meat-and-filler");

                return message;
            }
        }
    }
}
```

## 5. Create a NancyFX module to host the WebAPI
---
Create a new file (i.e BaconIpsumModule .cs) to hold the routes and actions for the Web API 
    
``` csharp
using Nancy;

namespace WebApplication
{
    public class BaconIpsumModule : NancyModule
    {
        public BaconIpsumModule(IBaconIpsumService baconIpsumService)
        {
            Get("/", args => "Super Duper Happy Path running on .NET Core");

            Get("/baconipsum", async args => await baconIpsumService.GenerateAsync());
        }
    }
}
```
    
The first route returns a string and the second one calls our service and returns a **BaconIpsumMessage** instance (And yes! <a href="http://nancyfx.org/" target="_blank">NancyFX</a> supports async too!).

Also note that the module requires an instance of **IBaconIpsumService** in the constructor but don't worry because **<a href="http://nancyfx.org/" target="_blank">NancyFX</a>** will inject it for you without any special configuration or registration!
            
## 6. Run and test the application
---

Run the following commands 
          
``` powershell
dotnet restore
dotnet run
```

Navigate to <a href="http://localhost:5000">http://localhost:5000</a> and the browser should show the message: **Super Duper Happy Path running on .NET Core**
      
Navigate to <a href="http://localhost:5000/baconipsum">http://localhost:5000/baconipsum</a> and because of automatic content navigation **<a href="http://nancyfx.org/" target="_blank">NancyFX</a>** will try to find a view to display the **BaconIpsumMessage** which we didn't register and therefore results in an internal server error:
      
<a href="/wp-content/uploads/2017/01/nancy500.jpg"><img src="/wp-content/uploads/2017/01/nancy500.jpg" alt="" class="alignleft wp-image-7141" srcset="/wp-content/uploads/2017/01/nancy500.jpg 1635w, /wp-content/uploads/2017/01/nancy500-300x86.jpg 300w, /wp-content/uploads/2017/01/nancy500-768x221.jpg 768w, /wp-content/uploads/2017/01/nancy500-1024x295.jpg 1024w, /wp-content/uploads/2017/01/nancy500-250x72.jpg 250w" sizes="(max-width: 1635px) 100vw, 1635px" /></a>
      
Now call the same url: <a href="http://localhost:5000/baconipsum">http://localhost:5000/baconipsum</a> using postman and you'll get the expected json:
      
<a href="/wp-content/uploads/2017/01/nancyPostman.jpg"><img src="/wp-content/uploads/2017/01/nancyPostman.jpg" alt="" class="alignleft wp-image-7151" srcset="/wp-content/uploads/2017/01/nancyPostman.jpg 2375w, /wp-content/uploads/2017/01/nancyPostman-300x119.jpg 300w, /wp-content/uploads/2017/01/nancyPostman-768x305.jpg 768w, /wp-content/uploads/2017/01/nancyPostman-1024x407.jpg 1024w, /wp-content/uploads/2017/01/nancyPostman-250x99.jpg 250w" sizes="(max-width: 2375px) 100vw, 2375px" /></a>
            
Get a copy of the code here: <a href="https://github.com/cmendible/dotnetcore.samples/tree/main/nancyfx.sample">https://github.com/cmendible/dotnetcore.samples/tree/main/nancyfx.sample</a>
            
Hope it helps!