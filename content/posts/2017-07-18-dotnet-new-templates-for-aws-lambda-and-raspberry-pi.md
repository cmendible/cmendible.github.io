---
author: Carlos Mendible
categories:
- dotnetcore
crosspost_to_medium: true
date: "2017-07-18T08:56:48Z"
description: dotnet new templates for AWS Lambda and Raspberry Pi
image: /wp-content/uploads/2017/07/dotnetcore.png
tags: ["AWS", "Lambda", "RaspberryPi", "Template"]
title: dotnet new templates for AWS Lambda and Raspberry Pi
---

After reading the following articles:

  * <a href="https://blogs.msdn.microsoft.com/dotnet/2017/04/02/how-to-create-your-own-templates-for-dotnet-new/" target="_blank">How to create your own templates for dotnet new</a>
  * <a href="https://rehansaeed.com/custom-project-templates-using-dotnet-new/" target="_blank">Custom Project Templates Using dotnet new</a>

and having a twitter conversation with <a href="https://social.msdn.microsoft.com/profile/Sayed-Ibrahim-Hashimi" target="_blank">Sayed-Ibrahim-Hashimi</a> I decided it was a nice idea to create some templates based on the code of some of my posts.

So here are the results:

## 1. Create a an ASP.NET Core Web API project ready for AWS lambda
---
  
* **Name:** CodeItYourself.WebAPI.AWS.Lambda
* **Type**: Web/API
* **Installation:** <code>dotnet new --install CodeItYourself.WebAPI.AWS.Lambda::*</code>
* **Usage:** <code>dotnet new webapilambda</code>
* **Related post:** [Deploy your ASP.NET Core Web API to AWS Lambda](/2017/07/04/deploy-your-asp-net-core-web-api-to-aws-lambda/)

## 2. Create an ASP.NET Core Web project for RaspberryPi
---

* **Name:** CodeItYourself.ASPNET.Raspberry 
* **Type**: Web
* **Installation:** <code>dotnet new --install CodeItYourself.ASPNET.Raspberry::*</code>
* **Usage:** <code>dotnet new webrpi</code>
* **Related post:** [Step by step: Running ASP.NET Core on Raspberry Pi](/2017/03/21/step-by-step-running-aspnet-core-on-raspberry-pi/)

You can clone the code for the templates <a href="https://github.com/cmendible/dotnetcore.templates" target="_blank">here</a>.

Hope it helps!