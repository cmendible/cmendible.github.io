---
author: Carlos Mendible
categories:
- dotnet
date: "2009-10-19T15:18:00Z"
description: Upgrade xVal.WebForms 0.1 to use xVal.1.0 (Validation Summary Support)
tags: ["ASP.Net", "Validation", "WebForms", "xVal"]
title: Upgrade xVal.WebForms 0.1 to use xVal.1.0 (Validation Summary Support)
---
Recently I found out about [xVal.WebForms](http://xvalwebforms.codeplex.com/) project at [CodePlex.com](http://codeplex.com). Its a good implementation of XVal for ASP.Net WebForms made by John Rummell.

I missed Validation Summary support so I upgraded John's code so it could work with XVal 1.0.

I submitted two patches modifying ModelValidator.cs and xVal.jquery.validate.js.

These changes are now part of the Trunk repository as you can see here:
  
[http://xvalwebforms.codeplex.com/SourceControl/PatchList.aspx](http://xvalwebforms.codeplex.com/SourceControl/PatchList.aspx)