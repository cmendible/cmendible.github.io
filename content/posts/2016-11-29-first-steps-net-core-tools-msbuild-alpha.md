---
author: Carlos Mendible
categories:
- dotnet
- dotnetcore
date: "2016-11-29T12:49:28Z"
description: First steps with .NET Core Tools MSBuild "alpha"
image: /wp-content/uploads/2016/11/netcoremsbuild.jpg
tags: ["msbuild"]
title: First steps with .NET Core Tools MSBuild "alpha"
url: /2016/11/29/first-steps-net-core-tools-msbuild-alpha/
---
On November 16th, Microsoft announced the <a href="https://blogs.msdn.microsoft.com/dotnet/2016/11/16/announcing-net-core-tools-msbuild-alpha/?Wt.mc_id=DX_MVP8656" target="_blank">.NET Core Tools MSBuild"alpha"</a>. I've been developing .Net Core applications with Visual Studio Code for a while now, and I needed to try the new tooling.

In this post I'll show you which were my **First steps with .NET Core Tools MSBuild "alpha"**

## 1. Installing .NET Core SDK 1.0 Preview 3 build 004056
---
The first step was to install the new tools from: <a href="https://github.com/dotnet/core/blob/master/release-notes/preview3-download.md" target="_blank">.NET Core SDK 1.0 Preview 3 build 004056</a><br />

## 2. Creating a Sample Console Application
---
Open a command prompt and run the following commands 

``` powershell 
    md test.msbuild
    cd test.msbuild
    dotnet new
    code .
```

Surprise Code shows two files and one is **test.msbuild.csproj**. Bye bye **project.json**!!!


## 3. There is no Intellisense for csproj files
---
This will make things difficult for a while but right now you have to forget about Intellisense and Autocomplete on your **csproj** file, which makes adding references a real pain.

## 4. The csproj file does not target .Net Core 1.1 by default
---
You'll have to edit the **csproj** file to target .Net Core 1.1 (lines 6 and 16)
    
``` xml 
<Project ToolsVersion="15.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <Import Project="$(MSBuildExtensionsPath)\$(MSBuildToolsVersion)\Microsoft.Common.props" />
  
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>netcoreapp1.1</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <Compile Include="**\*.cs" />
    <EmbeddedResource Include="**\*.resx" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NETCore.App">
      <Version>1.1.0</Version>
    </PackageReference>
    <PackageReference Include="Microsoft.NET.Sdk">
      <Version>1.0.0-alpha-20161104-2</Version>
      <PrivateAssets>All</PrivateAssets>
    </PackageReference>
  </ItemGroup>
  
  <Import Project="$(MSBuildToolsPath)\Microsoft.CSharp.targets" />
</Project>
```
## 5. Create launch.json
---
Hit F5 and select .NET Core. Visual Studio Code will create a **launch.json** file and because the tooling is not yet fully supported you'll have to edit the file manually with the following values 
    
``` json 
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": ".NET Core Launch (console)",
            "type": "coreclr",
            "request": "launch",
            "preLaunchTask": "build",
            "program": "${workspaceRoot}/bin/Debug/netcoreapp1.1/test.msbuild.dll",
            "args": [],
            "cwd": "${workspaceRoot}",
            "stopAtEntry": false,
            "externalConsole": false
        },
        {
            "name": ".NET Core Attach",
            "type": "coreclr",
            "request": "attach",
            "processId": "${command.pickProcess}"
        }
    ]
}
```
## 6. Create task.json
---
Hit F5 again and select .NET Core. Visual Studio Code will create a **task.json** file which you won't have to modify.
    
``` json 
{
    // See https://go.microsoft.com/fwlink/?LinkId=733558
    // for the documentation about the tasks.json format
    "version": "0.1.0",
    "command": "dotnet",
    "isShellCommand": true,
    "args": [],
    "tasks": [
        {
            "taskName": "build",
            "args": [ ],
            "isBuildCommand": true,
            "showOutput": "silent",
            "problemMatcher": "$msCompile"
        }
    ]
}
```
## 7. Restore packages
---
Run the following command from the Visual Studio Code terminal or the command prompt from step 1
    
``` powershell 
    dotnet restore
```
    
Visual Studio Code won't prompt automatically to restore so failing to run this step manually will prevent you from building the application.
      
## 8. Run and debug the application
---    
Place a break point in line 7 of the **Program.cs** file
          
``` powershell 
using System;

class Program
{
    static void Main(string[] args)
    {
        Console.WriteLine("Hello World!");
    }
}
```
          
Hit F5 and the program should stop at the break point and you are good to go!
            
## 9. Side by side with project.json tooling (no migration)
---     
To be able to run my applications based on the **project.json** tools I had to add a **global.json** file in the root folder, with the following content 
                
``` json 
{
    "sdk": {
        "version": "1.0.0-preview2-1-003177"
    }
}
```
 
If you are not sure of which versions you have installed, just run the following powershell command:
                      
``` powershell 
ls $env:programfiles\dotnet\sdk
```
                
And change the value in the **global.json** file as desired.
                  
Hope it helps!           