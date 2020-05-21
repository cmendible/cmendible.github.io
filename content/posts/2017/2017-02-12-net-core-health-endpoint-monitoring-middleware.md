---
author: Carlos Mendible
categories:
- dotnet
crosspost_to_medium: true
date: "2017-02-12T14:06:42Z"
description: .NET Core Health Endpoint Monitoring Middleware
images: ["/wp-content/uploads/2017/02/pipeline.png"]
tags: ["HealthEndpoint", "Middleware", "Monitoring"]
title: .NET Core Health Endpoint Monitoring Middleware
url: /2017/02/12/dotnet-core-health-endpoint-monitoring-middleware/
---
Today I'll show a simple example of how to create a **.Net Core Health Endpoint Monitoring <a href="https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware" target="_blank">Middleware</a>**.

If you need to know about the Health Endpoint Monitoring Pattern check: <a href="https://msdn.microsoft.com/en-us/library/dn589789.aspx" target="_blank">https://msdn.microsoft.com/en-us/library/dn589789.aspx</a>

## 1. Create the application
---
  
Open a command prompt and run 
    
``` powershell
    md service.status.middleware
    cd service.status.middleware
    dotnet new
    dotnet restore
    code .
```

## 2. Add the middleware ServiceStatusMiddleware class
---

Create a Middleware folder in your project and add a ServiceStatusMiddleware.cs file with the following contents 
    
``` csharp
namespace WebApplication
{
    using System;
    using System.Net;
    using System.Threading.Tasks;
    using Microsoft.AspNetCore.Builder;
    using Microsoft.AspNetCore.Http;
    using Newtonsoft.Json;

    /// <summary>
    /// Service Status Middleware used to check the Health of your service.
    /// </summary>
    public class ServiceStatusMiddleware
    {
        /// <summary>
        /// Next request RequestDelegate
        /// </summary>
        private readonly RequestDelegate next;

        /// <summary>
        /// Health check function.
        /// </summary>
        private readonly Func<Task<bool>> serviceStatusCheck;

        /// <summary>
        /// ServiceStatus enpoint path 
        /// </summary>
        private static readonly PathString statePath = new PathString("/_check");

        /// <summary>
        /// Constructor
        /// </summary>
        /// 
        /// 
        public ServiceStatusMiddleware(RequestDelegate next, Func<Task<bool>> serviceStatusCheck)
        {
            this.next = next;
            this.serviceStatusCheck = serviceStatusCheck;
        }

        /// <summary>
        /// Where the middleware magic happens
        /// </summary>
        /// 
        /// <returns>Task</returns>
        public async Task Invoke(HttpContext httpContext)
        {
            // If the path is different from the statePath let the request through the normal pipeline.
            if (!httpContext.Request.Path.Equals(statePath))
            {
                await this.next.Invoke(httpContext);
            }
            else
            {
                // If the path is statePath call the health check function.
                await CheckAsync(httpContext);
            }
        }

        /// <summary>
        /// Call the health check function and set the response.
        /// </summary>
        /// 
        /// <returns>Task</returns>
        private async Task CheckAsync(HttpContext httpContext)
        {
            if (await this.serviceStatusCheck().ConfigureAwait(false))
            {
                // Service is available.
                await WriteResponseAsync(httpContext, HttpStatusCode.OK, new ServiceStatus(true));
            }
            else
            {
                // Service is unavailable.
                await WriteResponseAsync(httpContext, HttpStatusCode.ServiceUnavailable, new ServiceStatus(false));
            }
        }

        /// <summary>
        /// Writes a response of the Service Status Check.
        /// </summary>
        /// 
        /// 
        /// 
        /// <returns>Task</returns>
        private Task WriteResponseAsync(HttpContext httpContext, HttpStatusCode httpStatusCode, ServiceStatus serviceStatus)
        {
            // Set content type.
            httpContext.Response.Headers["Content-Type"] = new[] { "application/json" };

            // Minimum set of headers to disable caching of the response.
            httpContext.Response.Headers["Cache-Control"] = new[] { "no-cache, no-store, must-revalidate" };
            httpContext.Response.Headers["Pragma"] = new[] { "no-cache" };
            httpContext.Response.Headers["Expires"] = new[] { "0" };

            // Set status code.
            httpContext.Response.StatusCode = (int)httpStatusCode;

            // Write the content.
            var content = JsonConvert.SerializeObject(serviceStatus);
            return httpContext.Response.WriteAsync(content);
        }
    }

    /// <summary>
    /// ServiceStatus to hold the response. 
    /// </summary>
    public class ServiceStatus
    {
        public ServiceStatus(bool available)
        {
            Available = available;
        }

        /// <summary>
        /// Tells if the service is available
        /// </summary>
        /// <returns>True if the service is available</returns>
        public bool Available { get; }
    }

    /// <summary>
    /// Service Status Middleware Extensions
    /// </summary>
    public static class ServiceStatusMiddlewareExtensions
    {
        public static IApplicationBuilder UseServiceStatus(
          this IApplicationBuilder app,
          Func<Task<bool>> serviceStatusCheck)
        {
            app.UseMiddleware<ServiceStatusMiddleware>(serviceStatusCheck);

            return app;
        }
    }

}
```

## 3. Add the middleware to the pipeline
---
  
In the **Startup** class find the **app.AddMvc** line and add the following just above 
    
``` csharp
// Call the ServiceStatus middleware with a random failing function. 
// Feel free to replace the function with any check you need.
Random random = new Random();
app.UseServiceStatus(() => 
{
    return Task.FromResult(random.Next(10) != 1);
});
```

## 4. Run the application
---
Open a command prompt and run 
    
``` powershell
    dotnet run
```
Browse to: <a href="http://localhost:5000/_check" target="_blank">http://localhost:5000/_check</a> and your browser should show

```json"
{"Available":true}
```
or

``` json
{"Available":false}
```
    
depending on the result of the random function we added to the middleware.
      
Checkout <a href="https://github.com/lurumad/aspnetcore-health" target="_blank">AspNetCore.Health</a> for nice and extensible implementation of the pattern.
  
Also find a copy of the code here: <a href="https://github.com/cmendible/dotnetcore.samples/tree/master/service.status.middleware">https://github.com/cmendible/dotnetcore.samples/tree/master/service.status.middleware</a>
  
Hope it helps!