---
author: Carlos Mendible
categories:
- dotNetCore
crosspost_to_medium: true
date: "2018-04-05T07:21:39Z"
description: .NET Core, Code Analysis and StyleCop with dotnet-addcasa
image: /wp-content/uploads/2017/07/dotnetcore.png
published: true
# tags: CodeAnalysis FxCop StyleCop
title: Adding Code Analysis and StyleCop checks to .NET Core projects with dotnet-addcasa
url: /2018/04/05/adding-code-analysis-and-stylecop-checks-to-net-core-projects-with-dotnet-addcasa/
---

Today I'll show you how to use [dotnet-addcasa](https://github.com/cmendible/dotnet-addcasa): a .NET Core global tool to add CodeAnalysis and Stylecop checks to your projects.

If you want to manually add those checks or understand the tool internals check my post: [.NET Core, Code Analysis and StyleCop]({{ site.baseurl }}{% post_url 2017-08-24-net-core-code-analysis-and-stylecop %})

## 1. Prerequisites
---

You'll need [.NET Core SDK 2.1.300-preview1](https://www.microsoft.com/net/download/dotnet-core/sdk-2.1.300-preview1) installed.

## 2. Install the dotnet-addcasa .NET Core tool
---

To install [dotnet-addcasa](https://github.com/cmendible/dotnet-addcasa) as a global tool run:

``` powershell
dotnet tool install -g dotnet-addcasa
```

## 3. Add Code Analysis and StyleCop checks to your projects
---

To add CodeAnalysis and StyleCop to all your projects run the following command from the root folder of your solution or workspace.

``` powershell
dotnet addcasa
```

The tool is a work in progress so feel free to contribute [here](https://github.com/cmendible/dotnet-addcasa)

Hope it helps!
