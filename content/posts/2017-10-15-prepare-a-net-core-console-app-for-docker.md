---
author: Carlos Mendible
categories:
- dotnetcore
crosspost_to_medium: true
date: "2017-10-15T12:16:08Z"
description: The following code shows you the structure of the console app we created
  and will help you Prepare a .Net Core Console App for Docker
image: /wp-content/uploads/2017/07/dotnetcore.png
tags: ["Docker", "Sidecar", "Container"]
title: Prepare a .Net Core Console App for Docker
---
Last week I had the luck to attend the <a href="https://www.microsoftevents.com/profile/form/index.cfm?PKformID=0x2564678abcd" rel="noopener" target="_blank">Microsoft Azure OpenHack in Amsterdam</a>. We spent two and a half days learning a lot about kubernetes, Azure Container Services, Azure Container Registry, Azure OMS and Minecraft! 

In one of the challenges we decided to implement a <a href="https://docs.microsoft.com/en-us/azure/architecture/patterns/sidecar" rel="noopener" target="_blank">sidecar container</a> for logging purposes. So using .NET Core we created a console application with proper handling of the"Control+C" and"Control+Break" key shortcuts.

The following code shows you the structure of the console app we created and will help you **Prepare a .Net Core Console App for Docker**

``` csharp
using System;
using System.Threading;
using System.Threading.Tasks;

namespace docker.controlc
{
    class Program
    {
        // AutoResetEvent to signal when to exit the application.
        private static readonly AutoResetEvent waitHandle = new AutoResetEvent(false);

        static void Main(string[] args)
        {
            // Fire and forget
            Task.Run(() =>
            {
                var random = new Random(10);
                while (true)
                {
                    // Write here whatever your side car applications needs to do.
                    // In this sample we are just writing a random number to the Console (stdout)
                    Console.WriteLine($"Loop = {random.Next()}");

                    // Sleep as long as you need.
                    Thread.Sleep(1000);
                }
            });

            // Handle Control+C or Control+Break
            Console.CancelKeyPress += (o, e) =>
            {
                Console.WriteLine("Exit");

                // Allow the manin thread to continue and exit...
                waitHandle.Set();
            };

            // Wait
            waitHandle.WaitOne();
        }
    }
}
```

Note that we are handling the Console.CancelKeyPress because Docker does not behave as expected if you use the typical Console.ReadKey method to make the application run until a key is pressed. 

Get the code [here](https://github.com/cmendible/dotnetcore.samples/tree/master/docker.controlc).

Hope it helps!