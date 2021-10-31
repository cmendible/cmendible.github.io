---
author: Carlos Mendible
categories:
- dotnet
date: "2017-05-30T11:58:00Z"
description: Start with Elasticssearch, Kibana and ASP.NET Core
images: ["/wp-content/uploads/2017/05/datalake.png"]
tags: ["aspnetcore", "docker", "elastic search", "kibana", "serilog"]
title: Start with Elasticssearch, Kibana and ASP.NET Core
url: /2017/05/30/start_with_elasticsearch_kibana_and_aspnet_core/
---
You want to **Start with <a href="https://www.elastic.co/products/elasticsearch" target="_blank">Elasticssearch</a>, <a href="https://www.elastic.co/products/kibana" target="_blank">Kibana</a> and ASP.NET Core** and also want to do it fast? Let's use Docker and find out how easy it can be:

## 1. Create a folder for your new project
---

Open a command prompt an run 
    
``` powershell
mkdir aspnet.elk.sample
cd aspnet.elk.sample
```
## 2. Create a new ASP.NET Core project
--- 

``` powershell
dotnet new mvc
```

## 3. Add the following Serilog packages
---
You´ll send the logs to ElasticSearch using Serilog:

    
``` powershell
dotnet add package Serilog -v 2.4.0
dotnet add package Serilog.Extensions.Logging -v 1.4.0
dotnet add package Serilog.Sinks.ElasticSearch -v 5.1.0
dotnet restore
```

## 4. Replace the contents of the Startup.cs file
---
Lines 27 and 66 are key to enable Serilog and the Elasticsearch sink:

    
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
using Serilog;
using Serilog.Events;
using Serilog.Sinks.Elasticsearch;

namespace aspnet.elk.sample
{
    public class Startup
    {
        public Startup(IHostingEnvironment env)
        {
            var builder = new ConfigurationBuilder()
                .SetBasePath(env.ContentRootPath)
                .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
                .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true)
                .AddEnvironmentVariables();

            // Create Serilog Elasticsearch logger
            Log.Logger = new LoggerConfiguration()
               .Enrich.FromLogContext()
               .MinimumLevel.Debug()
               .WriteTo.Elasticsearch().WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri("http://localhost:9200"))
               {
                   MinimumLogEventLevel = LogEventLevel.Verbose,
                   AutoRegisterTemplate = true,
               })
               .CreateLogger();

            Configuration = builder.Build();
        }

        public IConfigurationRoot Configuration { get; }

        // This method gets called by the runtime. Use this method to add services to the container.
        public void ConfigureServices(IServiceCollection services)
        {
            // Add framework services.
            services.AddMvc();
        }

        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
        {
            // loggerFactory.AddConsole(Configuration.GetSection("Logging"));
            loggerFactory.AddDebug();

            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
                app.UseBrowserLink();
            }
            else
            {
                app.UseExceptionHandler("/Home/Error");
            }

            // Add serilog
            loggerFactory.AddSerilog();

            app.UseStaticFiles();

            app.UseMvc(routes =>
            {
                routes.MapRoute(
                    name: "default",
                    template: "{controller=Home}/{action=Index}/{id?}");
            });
        }
    }
}
```

## 5. Use Docker to start Elasticsearch and Kibana
---
You'll need to add the following address range to your **docker unsafe registry**: **172.19.0.2:9200**

Run the following commands:

    
``` powershell
docker pull nshou/elasticsearch-kibana
docker run -d -p 9200:9200 -p 5601:5601 nshou/elasticsearch-kibana
```
    
It will take a while but you'll get a working Elasticsearch + Kibana installation.
      
## 6. Run the program and navigate
Run the program 
          
``` powershell
dotnet run
```

And navigate through some of the pages: <a href="http://localhost:5000" target="_blank">http://localhost:5000</a>
            
## 7. Setup Kibana and search your logs
---            
Open <a href="http://localhost:5601" target="_blank">http://localhost:5601</a> and configure the Index Pattern with the default values (Click Create at the bottom of the page).

Now click Discover on the side bar and start searching your logs. Enjoy!
            
Get the code and related files here: <a href="https://github.com/cmendible/dotnetcore.samples/tree/main/aspnet.elk.sample"  target="_blank">https://github.com/cmendible/dotnetcore.samples/tree/main/aspnet.elk.sample</a>     
        
Hope it helps!