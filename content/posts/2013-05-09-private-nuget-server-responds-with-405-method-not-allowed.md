---
author: Carlos Mendible
categories:
- dotNet
date: "2013-05-09T10:31:14Z"
description: Private Nuget Server responds with (405) Method Not Allowed while pushing
  or deleting a package
tags: ["IIS", "NuGet"]
title: Private Nuget Server responds with (405) Method Not Allowed while pushing or
  deleting a package
---
So you've deployed your own private **NuGet** server to **IIS** but you get this annoying exception trying to push packages:

**(405) Method Not Allowed**

Well as stated in many web pages removing **WebDAVModule** does fix the issue, but **be sure you add the configuration in the right section!** (not in the **elmah** section) or the fix wont work!

``` html
<configuration>
  <system.webServer>
    <validation validateIntegratedModeConfiguration="false" />
    <modules runAllManagedModulesForAllRequests="true">
      <remove name="WebDAVModule" />
    </modules>
    ...
```