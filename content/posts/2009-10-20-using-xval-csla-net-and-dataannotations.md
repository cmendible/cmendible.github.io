---
author: Carlos Mendible
categories:
- dotnet
date: "2009-10-20T14:59:00Z"
description: Using xVal, CSLA.NET and DataAnnotations.
tags: ["ASP.Net", "CSLA", "WebForms", "xVal"]
title: Using xVal, CSLA.NET and DataAnnotations.
url: /2009/10/20/using-xval-csla-net-and-dataannotations/
---
I've been working for years with CSLA to create business layer objects. Recently I started using XVal 1.0 and XVal.WebForms 0.1 for client side validation.

I started enforcing our rules as DataAnnotations and also in CSLA AddBusinessRules() method. So if a property of type string was required I used a StringLength DataAnnotation and also a CSLA.Validation.CommonRules.StringMaxLength rule in our business object.

I did not like that solution and today I found out about [CSLA.Net 3.8 Beta 1](http://www.lhotka.net/weblog/CSLANET38Beta1Available.aspx) which now supports DataAnnotations. (Find more about this [here](http://www.lhotka.net/weblog/PermaLink,guid,7b05be46-15bf-4388-95b6-14f6d7af08e5.aspx)

This is a very important enhancement to CSLA, made by Rocky. I tested it without any issues so now here at [HexaSystems](http://www.hexasystems.com/index.php) we don't longer need to enforce twice these type of rules.

Note tha this CSLA version depends on assembly: System.Windows.Interactivity which comes as part of [Microsoft Expression Blend 3 SDK](http://www.microsoft.com/DOWNLOADS/details.aspx?FamilyID=f1ae9a30-4928-411d-970b-e682ab179e17&displaylang=en)