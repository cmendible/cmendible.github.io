---
author: Carlos Mendible
categories:
- dotnet
date: "2009-10-26T09:10:00Z"
description: xVal.WebForms, Validation Groups and CausesValidation
tags: ["aspnet", "xval"]
title: xVal.WebForms, Validation Groups and CausesValidation
url: /2009/10/26/xval-webforms-validation-groups-and-causesvalidation/
---
In one of our recent projects we needed a behavior like the ASP.Net **ValidationGroups** and also be able to relay on the **CausesValidation** of all controls implementing **IButtonControl** interface control.

So the first issue I faced was that jquery-validate plugin version 1.5.1 (The one that xVal.WebForms uses) does not support grouping in an ASP.Net way. I found the solution to this first problem in this article: <http://plugins.jquery.com/node/7044>. I did some tests on **worldspawn** work and did not find any problem with the implementation.

The groups must be defined as in the following sample:

``` javascript
valgroups: {
  test: { buttons: [ " ], fields: [ ", " ] },
  foo: { buttons: [ " ], fields: [", " ] }
}
 ```

So the next issue was making xVal know about this valgroups options and passing it as a parameter to the patched version of jquery-validate. For this I patched again file xVal.jquery.validate.js adding the following lines:

``` javascript
if (options.valgroups) {
  validationOptions.valgroups = options.valgroups;
}
``` 

Then it was all about changing xVal.WebForms.ModelValidator control to render the needed javascript to suppress validation when controls have the CausesValidation == false, and render the groups options for xVal.

The complete patch can be found [here](http://xvalwebforms.codeplex.com/SourceControl/PatchList.aspx) ID: 4232

Enjoy.