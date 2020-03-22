---
author: Carlos Mendible
categories:
- dotnet
crosspost_to_medium: true
date: "2018-08-25T11:27:39Z"
description: Adding SourceLink to your .NET Core Library
image: /wp-content/uploads/2017/07/dotnetcore.png
published: true
tags: ["sourcelink", "debug"]
title: Adding SourceLink to your .NET Core Library
url: /2018/08/25/adding-sourcelink-to-your-net-core-library/
---

Last week I read this tweets from Maxime Rouiller ([@MaximRouiller](https://twitter.com/MaximRouiller)):

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Are you an MVP with a <a href="https://twitter.com/hashtag/dotnetcore?src=hash&amp;ref_src=twsrc%5Etfw">#dotnetcore</a> <a href="https://twitter.com/hashtag/nuget?src=hash&amp;ref_src=twsrc%5Etfw">#nuget</a> package? Are you looking for an easy blog post? I have something for you.</p>&mdash; Maxime Rouiller (@MaximRouiller) <a href="https://twitter.com/MaximRouiller/status/1032370596082900992?ref_src=twsrc%5Etfw">August 22, 2018</a></blockquote>

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Actually, you&#39;re already using it. <a href="https://t.co/N5IaY6TGqQ">https://t.co/N5IaY6TGqQ</a><br><br>That project is freaking cool. We need more package author to use it.</p>&mdash; Maxime Rouiller (@MaximRouiller) <a href="https://twitter.com/MaximRouiller/status/1032372247623696384?ref_src=twsrc%5Etfw">August 22, 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Then I found out that [SourceLink](https://github.com/dotnet/sourcelink) is a language-and source-control agnostic system for providing first-class source debugging experiences for binaries."

What that means is that, once you enable SourceLink, the users are able to step into your code without effort! from  both Visual Studio and Visual Studio Code which are ready to let you enjoy the experience.

This findings triggered my interest and started wondering on how could I add such a nice feature to the [netDumbster](https://github.com/cmendible/netDumbster) project.

Here is what I did:

## 1. Added the following properties to the project's **csproj**

``` xml
<PublishRepositoryUrl>true</PublishRepositoryUrl>
<EmbedUntrackedSources>true</EmbedUntrackedSources>
<AllowedOutputExtensionsInPackageBuildOutputFolder>$(AllowedOutputExtensionsInPackageBuildOutputFolder);.pdb</AllowedOutputExtensionsInPackageBuildOutputFolder>
```

## 2. Added the following **ItemGroup** to the project's **csproj**

``` xml
<ItemGroup>
  <PackageReference Include="Microsoft.SourceLink.GitHub" Version="1.0.0-beta-63127-02" PrivateAssets="All"/>
</ItemGroup>
```

**Note**: this Package Reference depends on the source control system.

## 3. Finally built the nuget package as usual

``` shell
dotnet pack -c release
```

Believe me that was all!!! So then I tested the new package and features with a simple console application and Visual Studio Code:

## 1. Created a new console project

``` shell
mkdir SourceLinkTest
cd SourceLinkTest
dotnet new console
dotnet add package netDumbster -v 2.0.0.3
dotnet restore
```

## 2. Replaced the contents of Program.cs with the following code

``` csharp
using System;
using netDumbster.smtp;
using netDumbster.smtp.Logging;

namespace a
{
    class Program
    {
        static void Main(string[] args)
        {
            LogManager.GetLogger = type => new ConsoleLogger(type);
            SimpleSmtpServer.Start();
        }
    }
}
```

## 3. Generated Assets for Build and Debug for .NET

Inside Visual Studio Code and from the Command Palette invoke the **.NET: Generate Assets for Build and Debug** command so it creates the **launch.json** and **tasks.json** files inside the **.vscode** folder.

## 4. Replace the contents of **launch.json** with the following configuration

``` json
{
   // Use IntelliSense to find out which attributes exist for C# debugging
   // Use hover for the description of the existing attributes
   // For further information visit https://github.com/OmniSharp/omnisharp-vscode/blob/master/debugger-launchjson.md
   "version": "0.2.0",
   "configurations": [
        {
            "name": ".NET Core Launch (console)",
            "type": "coreclr",
            "request": "launch",
            "preLaunchTask": "build",
            // If you have changed target frameworks, make sure to update the program path.
            "program": "${workspaceFolder}/bin/Debug/netcoreapp2.1/SourceLinkTest.dll",
            "args": [],
            "cwd": "${workspaceFolder}",
            // For more information about the 'console' field, see https://github.com/OmniSharp/omnisharp-vscode/blob/master/debugger-launchjson.md#console-terminal-window
            "console": "internalConsole",
            "stopAtEntry": false,
            "internalConsoleOptions": "openOnSessionStart",
            "justMyCode": false,
            "suppressJITOptimizations": true
        },
        {
            "name": ".NET Core Attach",
            "type": "coreclr",
            "request": "attach",
            "processId": "${command:pickProcess}"
        }
    ,]
}
```

**Note**: the configuration disables **justMyCode** and enables **suppressJITOptimizations**.

## 5. Enjoyed the SourceLink debugging experience

I added a breakpoint in line 12 of **Program.cs** and started debugging. Once the debugger hit the breakpoint I pressed F11 to "Step Into" the netDumbster code. And surprise! I was in!!!

![image shows the debugger inside netDumbster's code](/assets/img/posts/netDumbster_SourceLink.png)

Now let me ask you something: What are you waiting to get out there and add [SourceLink](https://github.com/dotnet/sourcelink) to your existing library?

Hope you enjoyed the ride!