---
author: Carlos Mendible
categories:
- dotnet
- azure
crosspost_to_medium: true
date: "2020-02-09T00:00:00Z"
description: 'Dapr: Debugging .NET Core with Visual Studio Code'
image: /assets/img/posts/aks.png
published: true
tags: ["dapr", "code", "debug"]
title: 'Dapr: Debugging .NET Core with Visual Studio Code'
url: /2020/02/09/dapr-debugging-dotnetcore-with-visual-studio-code/
---

So you are new to [Dapr](https://dapr.io) and you are trying to understand how it works with you .NET Core application. You already tried launching your app with the Dapr CLI and then you find yourself wondering on how to debug the mix with Visual Studio Code.

Well, follow this simple steps and you'll be ready:

## Modify launch.json

Change the **preLaunchTask** so instead it calls a task named **daprd-debug** instead of **build**

Your **launch.json** file should look like this:
 
``` json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": ".NET Core Launch (web)",
            "type": "coreclr",
            "request": "launch",
            "preLaunchTask": "daprd-debug",
            "program": "${workspaceFolder}/bin/Debug/netcoreapp3.1/<your assembly name>.dll",
            "args": [],
            "cwd": "${workspaceFolder}",
            "stopAtEntry": false,
            "serverReadyAction": {
                "action": "openExternally",
                "pattern": "^\\s*Now listening on:\\s+(https?://\\S+)"
            },
            "env": {
                "ASPNETCORE_ENVIRONMENT": "Development"
            },
            "sourceFileMap": {
                "/Views": "${workspaceFolder}/Views"
            }
        },
        {
            "name": ".NET Core Attach",
            "type": "coreclr",
            "request": "attach",
            "processId": "${command:pickProcess}"
        }
    ]
}
```

## Modify tasks.json

A new task to **tasks.json** and name it **daprd-debug** or whatever name you choosed in te previous step.

Your **tasks.json** should look like this:
 
``` json
{
    "version": "2.0.0",
    "tasks": [
        // Ommited tasks.
        // ...
        {
            "label": "daprd-debug",
            "command": "daprd",
            "args": [
                "-dapr-id",
                "routing",
                "-app-port",
                "5000"
            ],
            "isBackground": true,
            "problemMatcher": {
                "pattern": [
                    {
                        "regexp": ".",
                        "file": 1,
                        "location": 2,
                        "message": 3
                    }
                ],
                "background": {
                    "beginsPattern": "^.*starting Dapr Runtime.*",
                    "endsPattern": "^.*waiting on port.*"
                }
            },
            "dependsOn": "build"
        }
    ]
}
```

**Note:** the new task depends on the **build** task.

## Hit F5 and enjoy

Hit F5 and enjoy debugging your "Daprized" .NET Core application.

You can download a working sample from this [repo](https://github.com/cmendible/dotnetcore.samples/tree/master/dapr.debug.vscode/).
