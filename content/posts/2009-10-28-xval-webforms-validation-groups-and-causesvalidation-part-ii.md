---
author: Carlos Mendible
categories:
- dotNet
date: "2009-10-28T15:22:00Z"
description: xVal.WebForms, Validation Groups and CausesValidation Part II
tags: ["ASP.Net", "CausesValidation", "ValidationGroups", "WebForms", "xVal"]
title: xVal.WebForms, Validation Groups and CausesValidation Part II
---
A friend came with two issues concerning my recent post about xVal.webForms ValidationGroups and CausesValidation support.

1.- Supress Validation was being rendered as many times as ModelValidators where in the page.
  
2.- When more than one ModelValidator control was in place, xVal.AttachValidator script options could be incomplete, specifically the valgroups, cause the loop through page collections was done in the OnInit method of the controls before the page control tree was complete.

So I made the changes in a way that all the code called from ModelValidator's render methods is now called in the page's PreRenderComplete event to which the control subscribes in its OnInit stage.

This changes allows to render only one call to SupressValidation if needed, and ensures that all calls to xVal.AttachValidator method has the complete set of valgroups added.

The patch file can be downloaded [here](http://xvalwebforms.codeplex.com/SourceControl/PatchList.aspx) Patch ID: 4250