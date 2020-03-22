---
author: Carlos Mendible
categories:
- dotnet
date: "2016-07-04T08:20:15Z"
description: Add Swagger to your .NET Core Web API
image: /wp-content/uploads/2016/07/AddSwaggerHttp.png
tags: ["Swagger", "Swashbuckle", "WebAPI"]
title: Add Swagger to your .NET Core Web API
---
Last week <a href="https://www.microsoft.com/net/core" target="_blank">.NET Core</a> was released, and the first thing I tried to solve was how to **Add Swagger to your .NET Core Web API**:

## 1. Dependencies
---
At the time of writing you should add the following dependency to your **project.json** file 
    
``` csharp
    "Swashbuckle": "6.0.0-beta901"
```

## 2. Using statement
---
In your **Startup** class add the following using statement 
    
``` csharp
    using Swashbuckle.Swagger.Model;
```

## 3. Add Swagger as a service
---
In your **Startup** class add the following code to the **ConfigureServices** method 
    
``` csharp
    // This method gets called by the runtime. Use this method to add services to the container.
    public void ConfigureServices(IServiceCollection services)
    {
       ...

       services.AddSwaggerGen();
       services.ConfigureSwaggerGen(options =>
       {
           options.SingleApiVersion(new Info
           {
               Version = "v1",
               Title = "Your API title here",
               Description = "Your API description here",
               TermsOfService = "Your API terms of service here"
           });
       });
    }
```

## 4. Add Swagger to the HTTP request pipeline
---
And finally add the following lines to the **Configure** method of your **Startup** class 
    
``` csharp
    // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
    public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
    {
        ...

        app.UseSwagger();
        app.UseSwaggerUi();
    }
```

And that's it you are ready to go!

Hope it helps!