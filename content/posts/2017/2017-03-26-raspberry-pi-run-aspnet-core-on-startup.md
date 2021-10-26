---
author: Carlos Mendible
categories:
- dotnet
date: "2017-03-26T12:37:16Z"
description: 'Raspberry Pi: Run ASP.NET Core on Startup'
images: ["/wp-content/uploads/2017/03/windows_10_iot_core_launch.jpg"]
tags: ["ARM", "aspNetCore", "PowerShell", "RaspberryPi", "Windows10", "IoT"]
title: 'Raspberry Pi: Run ASP.NET Core on Startup'
url: /2017/03/26/raspberry-pi-run-aspnet-core-on-startup/
---
Last week I wrote: **[Step by step: Running ASP.NET Core on Raspberry Pi](https://carlos.mendible.com/2017/03/21/step-by-step-running-aspnet-core-on-raspberry-pi/)** and didn't have the time to write about running the application on startup.

After browsing for a while I found this great post: [Windows IoT Core: Running a PowerShell Script on Startup](https://microsoft.hackster.io/en-US/falafel-software/windows-iot-core-running-a-powershell-script-on-startup-0aa534) which showed me the way!

As a prerequisite read and run the sample provided here: **[Step by step: Running ASP.NET Core on Raspberry Pi](https://carlos.mendible.com/2017/03/21/step-by-step-running-aspnet-core-on-raspberry-pi/)**

Let's start:

## 1. Create startup.ps1
---
Create **startup.ps1** with the following contents 
    
``` powershell
Set-Location C:\publish\
.\aspnet.on.rpi.exe
```

## 2. Create startup.bat
---
Create **startup.bat** with the following contents 
    
``` powershell
powershell -command "C:\startup.ps1"
```

## 3. Copy the files to your Raspberry
---
Connect to Raspberry using powershell, start the ftp server
    
``` powershell
Enter-PSSession -ComputerName <Raspberry IP> -Credential <Raspberry IP>\Administrator
start C:\Windows\System32\ftpd.exe
```

Open the **File Explorer** ftp://<TARGET_DEVICE> and copy both **startup.ps1** and **startup.cmd** to your Raspberry
      
## 4. Schedule the command to run on Startup
---      
Connect to Raspberry using powershell and run
     
          
``` powershell
Enter-PSSession -ComputerName <Raspberry IP> -Credential <Raspberry IP>\Administrator
schtasks /create /tn "Startup Web" /tr c:\Startup.bat /sc onstart /ru SYSTEM
```
      
## 5. Restart and verify
---      
Restart your Raspberry and after a bit your ASP.NET Core app should be up and running.
         
Get the code here: <a href="https://github.com/cmendible/dotnetcore.samples/tree/main/aspnet.on.rpi.startup">https://github.com/cmendible/dotnetcore.samples/tree/main/aspnet.on.rpi.startup</a>
  
Hope it helps!
  