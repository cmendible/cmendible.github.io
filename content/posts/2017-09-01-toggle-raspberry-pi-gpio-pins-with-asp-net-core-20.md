---
author: Carlos Mendible
categories:
- dotnetcore
crosspost_to_medium: true
date: "2017-09-01T09:39:08Z"
description: Toggle Raspberry Pi GPIO Pins with ASP.NET Core 2.0
image: /wp-content/uploads/2017/09/IMG_20170901_111428_01.jpg
# tags: ARM aspNetCore GPIO RaspberryPi Raspbian
title: Toggle Raspberry Pi GPIO Pins with ASP.NET Core 2.0
---
Today I'll show you how to **Toggle Raspberry Pi GPIO Pins with ASP.NET Core 2.0**.

First be aware of the following prerequisites:

  * **<a href="https://www.microsoft.com/net/download/core" target="_blank">.NET Core 2.0 SDK</a>**
  * **A Raspberry Pi 3 Running Raspbian**
  * Install linux dependencies: sudo apt-get install curl libunwind8 gettext

Now let's start:

## 1. Create a folder for your new project
---
Open a command prompt an run: 
    
``` powershell
mkdir aspnet.webapi.rpi.gpio
cd aspnet.webapi.rpi.gpio
```

## 2. Create an ASP.NET Core Web API project
---
Create an ASP.NET Core Web API project with the following command:
    
``` powershell
dotnet new webapi
```

## 3. Add a reference to Unosquare.Raspberry.IO
---
``` powershell
dotnet add package Unosquare.Raspberry.IO
```

Now restore the packages:
    
``` powershell
dotnet restore
```
## 4. Create BlinkyController.cs
---
In the Controllers folder, create a BlinkyController.cs file with the following contents:

``` csharp
using Microsoft.AspNetCore.Mvc;
using Unosquare.RaspberryIO;
using Unosquare.RaspberryIO.Gpio;

namespace aspnet.webapi.rpi.gpio.Controllers
{
    [Route("api/[controller]")]
    public class BlinkyController : Controller
    {
        [HttpPost]
        public void Post([FromBody]bool isOn)
        {
            // Control GPIO pin 5
            var pin = Pi.Gpio.Pin05;
            pin.PinMode = GpioPinDriveMode.Output;
            pin.Write(!isOn);
        }
    }
}
```
    
Note: The sample toggles GPIO pin 5. 
      
## 5. Publish the application
---
Publish the application with the following commands: 
          
``` powershell
dotnet publish -c Release -r linux-arm
```
 
## 6. Copy the files to your Raspberry
---
Copy the contents of the folder **\aspnet.on.rpi\bin\Release\netcoreapp2.0\linux-arm\publish** to the Raspberry (i.e. /home/pi/publish)

**Note:** For this task I connect to the Raspberry via SFTP using <a href="https://winscp.net" target="_blank">WinSCP</a>
            
## 7. Run the application
---
Connect to Raspberry using ssh and then run the following commands:
               
``` powershell
sudo -i
cd /home/pi/publish
chmod +x aspnet.webapi.rpi.gpio
export ASPNETCORE_URLS="http://*:5000"
./aspnet.webapi.rpi.gpio
```
                
Note: Unosquare.Raspberry.IO must be run with elevated privileges.
                  
## 8. Call the Web API to Toggle the GPIO pin
---
Post data to the Web API to toggle the GPIO pin.

To turn it on:

``` powershell
curl -H "Content-Type: application/json" -d 'true' http://[ip of your raspberry]:5000/api/blinky
```
                      
To Turn it off:
``` powershell
curl -H "Content-Type: application/json" -d 'false' http://[ip of your raspberry]:5000/api/blinky
```

Get the code <a href="https://github.com/cmendible/dotnetcore.samples/tree/master/aspnet.webapi.rpi.gpio" target="_blank">here</a>.

Hope it helps!