---
title: dotnet new templates for AWS Lambda and Raspberry Pi
date: 2017-07-18T08:56:48+00:00
author: Carlos Mendible
layout: post
image: /wp-content/uploads/2017/07/dotnetcore.png
categories:
  - dotNetCore
tags:
  - AWS
  - dotnet new
  - Lambda
  - Raspberry Pi
  - template
---

After reading the following articles:

  * <a href="https://blogs.msdn.microsoft.com/dotnet/2017/04/02/how-to-create-your-own-templates-for-dotnet-new/" target="_blank">How to create your own templates for dotnet new</a>
  * <a href="https://rehansaeed.com/custom-project-templates-using-dotnet-new/" target="_blank">Custom Project Templates Using dotnet new</a>

and having a twitter conversation with <a href="https://social.msdn.microsoft.com/profile/Sayed-Ibrahim-Hashimi" target="_blank">Sayed-Ibrahim-Hashimi</a> I decided it was a nice idea to create some templates based on the code of some of my posts.

So here are the results:

## 1. Create a an ASP.NET Core Web API project ready for AWS lambda
---
  
**Name** | **Type** | **Installation** |  **Usage** | **Related Post**
--- | --- | --- | --- | ---
CodeItYourself.WebAPI.AWS.Lambda | Web/API | <code>dotnet new --install CodeItYourself.WebAPI.AWS.Lambda::*</code> | <code>dotnet new webapilambda</code> | <a href="/2017/07/04/deploy-your-asp-net-core-web-api-to-aws-lambda/" target="_blank">Deploy   

## 2. Create an ASP.NET Core Web project for RaspberryPi
---
**Name** | **Type** | **Installation** |  **Usage** | **Related Post**
--- | --- | --- | --- | ---
CodeItYourself.ASPNET.Raspberry | Web | <code>dotnet new --install CodeItYourself.ASPNET.Raspberry::*</code> | <code>dotnet new webrpi</code> | <a href="/2017/03/21/step-by-step-running-aspnet-core-on-raspberry-pi/" target="_blank">Step by step: Running ASP.NET Core on Raspberry Pi</a>

You can clone the code for the templates <a href="https://github.com/cmendible/dotnetcore.templates" target="_blank">here</a>.

Hope it helps!