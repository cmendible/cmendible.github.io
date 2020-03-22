---
author: Carlos Mendible
categories:
- dotnet
- dotnetcore
crosspost_to_medium: true
date: "2017-03-21T11:24:05Z"
description: 'Step by step: Running ASP.NET Core on Raspberry Pi'
image: /wp-content/uploads/2017/03/aspnetcorerpi.gif
tags: ["ARM", "aspNetCore", "RaspberryPi", "Windows10", "IoT"]
title: 'Step by step: Running ASP.NET Core on Raspberry Pi'
url: /2017/03/21/step-by-step-running-aspnet-core-on-raspberry-pi/
---
After reading <a href="https://github.com/dotnet/core/blob/master/samples/RaspberryPiInstructions.md" target="_blank">.NET Core on Raspberry Pi</a> and successfully running a console application on **Windows 10 IoT Core** on my **Raspberry Pi 3** I decided to write: **Step by step: Running ASP.NET Core on Raspberry Pi**.

First be aware of the following prerequisites:

  * **<a href="https://developer.microsoft.com/en-us/windows/iot/GetStarted" target="_blank">Windows 10 IoT Core</a>** I'm running Insider Preview v.10.0.15058.0
  * **<a href="https://github.com/dotnet/cli/tree/master" target="_blank">.NET Core 2.0 SDK</a>**

Now let's start:

## 1. Create a folder for your new project
---
Open a command prompt an run 
    
``` powershell
mkdir aspnet.on.rpi
cd aspnet.on.rpi
code .
```

## 2. Create a global.json file
---
To specify the correct sdk create a **global.json** with the following contents 
    
``` json
{
    "sdk": {
        "version": "2.0.0-preview1-005448"
    }
}
```

## 3. Create the ASP.NET Core project
---
Create the ASP.NET Core project with the following command:
    
``` powershell
dotnet new mvc
```

## 4. Modify the project file
---
Modify the **aspnet.on.rpi.csproj** to add the correct **OutputType**, **TargetFramework**, **RuntimeFrameworkVersion**and **RuntimeIdentifiers**
    
``` xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>netcoreapp2.0</TargetFramework>
    <RuntimeFrameworkVersion>2.0.0-beta-001776-00</RuntimeFrameworkVersion>
    <RuntimeIdentifiers>win8-arm</RuntimeIdentifiers>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore" Version="1.0.3" />
    <PackageReference Include="Microsoft.AspNetCore.Mvc" Version="1.0.2" />
    <PackageReference Include="Microsoft.AspNetCore.StaticFiles" Version="1.0.1" />
    <PackageReference Include="Microsoft.Extensions.Logging.Debug" Version="1.0.1" />
    <PackageReference Include="Microsoft.VisualStudio.Web.BrowserLink" Version="1.0.1" />
    <PackageReference Include="Libuv" Version="1.10.0-preview1-22036" />
  </ItemGroup>
</Project>
```

## 5. Add a Nuget.config file and restore packages
Create a **Nuget.config** file with the following contents:
    
``` xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="dotnet-core" value="https://dotnet.myget.org/F/dotnet-core/api/v3/index.json" />
  </packageSources>
</configuration>
```
    
Now restore the packages:
    
``` powershell
dotnet restore
```

## 6. Modify Program.cs
---
Replace the contents of the **Program.cs** file with the following code 
    
``` csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Hosting;

namespace aspnet.on.rpi
{
    public class Program
    {
        public static void Main(string[] args)
        {
            var host = new WebHostBuilder()
                .UseKestrel()
                .UseUrls("http://*:5000") // Using * to bind to all network interfaces
                .UseContentRoot(Directory.GetCurrentDirectory())
                .UseIISIntegration()
                .UseStartup<Startup>()
                .Build();

            host.Run();
        }
    }
}
```

## 7. Publish the application
---
Publish the application with the following commands 
    
``` powershell
dotnet publish -r win8-arm
```

<del datetime="2017-03-23T20:33:55+00:00">We need to publish for win7-arm as a workaround to copy the correct **libuv.dll** in the next step.</del>
      
8. Copy libuv.dll
---      
<del datetime="2017-03-23T20:33:55+00:00">Copy **libuv.dll** from **\aspnet.on.rpi\bin\Debug\netcoreapp2.0\win7-arm\publish** to **\aspnet.on.rpi\bin\Debug\netcoreapp2.0\win8-arm\publish**</del>
          
This step is no longer needed cause we added **libuv** as a dependency in the **csproj** file
      
## 9. Copy the files to your Raspberry
---      
Connect to Raspberry using PowerShell, start the ftp server and open port 5000 on the Raspberry
      
          
``` powershell
Enter-PSSession -ComputerName <Raspberry IP> -Credential <Raspberry IP>\Administrator
start C:\Windows\System32\ftpd.exe
netsh advfirewall firewall add rule name="Open Port 5000" dir=in action=allow protocol=TCP localport=5000
```
          
Open the **File Explorer** ftp://<TARGET_DEVICE> and copy the contents of **\aspnet.on.rpi\bin\Debug\netcoreapp2.0\win8-arm\publish** to a folder on your Raspberry (i.e. c:\publish).

## 10. Run the application
---            
Connect to Raspberry using PowerShell and run
                            
``` powershell
cd c:\publish\
.\aspnet.on.rpi.exe
```
You should be good to go and be able to browse on port 5000 of you RPi.
        
Get the code here: <a href="https://github.com/cmendible/dotnetcore.samples/tree/master/aspnet.on.rpi">https://github.com/cmendible/dotnetcore.samples/tree/master/aspnet.on.rpi</a>
        
Hope it helps!        