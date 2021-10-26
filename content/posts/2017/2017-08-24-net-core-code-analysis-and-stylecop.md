---
author: Carlos Mendible
categories:
- dotnet
date: "2017-08-24T17:01:39Z"
description: .NET Core, Code Analysis and StyleCop
images: ["/wp-content/uploads/2017/07/dotnetcore.png"]
tags: ["CodeAnalysis", "FxCop", "StyleCop"]
title: .NET Core, Code Analysis and StyleCop
url: /2017/08/24/dotnet-core-code-analysis-and-stylecop/
---
So now that .NET Core and .NET Standard 2.0 have been released some of you may be migrating applications or even creating new ones with it. As you progress you are starting to worry about the quality of your code so what you want is to at least check your code against design and style guidelines don't you? 

This post will show you how easy is to make **.NET Core, Code Analysis and StyleCop** play along together.

Here you go:

## 1. Create a folder for your new project
---
Open a command prompt an run:
    
``` powershell
mkdir codeanalysis
```
 
## 2. Create the project
---

``` powershell
cd codeanalysis
dotnet new console
```
## 3. Add the references for CodeAnalysis and StyleCop
---

``` powershell
dotnet add package Microsoft.CodeAnalysis.FxCopAnalyzers
dotnet add package StyleCop.Analyzers
dotnet restore
```

## 4. Add CodeAnalysisRuleSet property to the project
---
Open the file **codeanalysis.csproj** and add the following property inside a Property Group:

``` xml
<CodeAnalysisRuleSet>ca.ruleset</CodeAnalysisRuleSet>
```

or add a new Property Group:

``` xml
<PropertyGroup>
    <CodeAnalysisRuleSet>ca.ruleset</CodeAnalysisRuleSet>
</PropertyGroup>
```

## 5. Create the ca.ruleset file
--- 
Create a ca.ruleset file and replace it's content with the following xml:
 
``` xml
<?xml version="1.0" encoding="utf-8"?>
<RuleSet Name="Custom Rulset" Description="Custom Rulset" ToolsVersion="14.0">
    <Rules AnalyzerId="AsyncUsageAnalyzers" RuleNamespace="AsyncUsageAnalyzers">
        <Rule Id="UseConfigureAwait" Action="Warning" />
    </Rules>
    <Rules AnalyzerId="Microsoft.Analyzers.ManagedCodeAnalysis" RuleNamespace="Microsoft.Rules.Managed">
        <Rule Id="CA1001" Action="Warning" />
        <Rule Id="CA1009" Action="Warning" />
        <Rule Id="CA1016" Action="Warning" />
        <Rule Id="CA1033" Action="Warning" />
        <Rule Id="CA1049" Action="Warning" />
        <Rule Id="CA1060" Action="Warning" />
        <Rule Id="CA1061" Action="Warning" />
        <Rule Id="CA1063" Action="Warning" />
        <Rule Id="CA1065" Action="Warning" />
        <Rule Id="CA1301" Action="Warning" />
        <Rule Id="CA1400" Action="Warning" />
        <Rule Id="CA1401" Action="Warning" />
        <Rule Id="CA1403" Action="Warning" />
        <Rule Id="CA1404" Action="Warning" />
        <Rule Id="CA1405" Action="Warning" />
        <Rule Id="CA1410" Action="Warning" />
        <Rule Id="CA1415" Action="Warning" />
        <Rule Id="CA1821" Action="Warning" />
        <Rule Id="CA1900" Action="Warning" />
        <Rule Id="CA1901" Action="Warning" />
        <Rule Id="CA2002" Action="Warning" />
        <Rule Id="CA2100" Action="Warning" />
        <Rule Id="CA2101" Action="Warning" />
        <Rule Id="CA2108" Action="Warning" />
        <Rule Id="CA2111" Action="Warning" />
        <Rule Id="CA2112" Action="Warning" />
        <Rule Id="CA2114" Action="Warning" />
        <Rule Id="CA2116" Action="Warning" />
        <Rule Id="CA2117" Action="Warning" />
        <Rule Id="CA2122" Action="Warning" />
        <Rule Id="CA2123" Action="Warning" />
        <Rule Id="CA2124" Action="Warning" />
        <Rule Id="CA2126" Action="Warning" />
        <Rule Id="CA2131" Action="Warning" />
        <Rule Id="CA2132" Action="Warning" />
        <Rule Id="CA2133" Action="Warning" />
        <Rule Id="CA2134" Action="Warning" />
        <Rule Id="CA2137" Action="Warning" />
        <Rule Id="CA2138" Action="Warning" />
        <Rule Id="CA2140" Action="Warning" />
        <Rule Id="CA2141" Action="Warning" />
        <Rule Id="CA2146" Action="Warning" />
        <Rule Id="CA2147" Action="Warning" />
        <Rule Id="CA2149" Action="Warning" />
        <Rule Id="CA2200" Action="Warning" />
        <Rule Id="CA2202" Action="Warning" />
        <Rule Id="CA2207" Action="Warning" />
        <Rule Id="CA2212" Action="Warning" />
        <Rule Id="CA2213" Action="Warning" />
        <Rule Id="CA2214" Action="Warning" />
        <Rule Id="CA2216" Action="Warning" />
        <Rule Id="CA2220" Action="Warning" />
        <Rule Id="CA2229" Action="Warning" />
        <Rule Id="CA2231" Action="Warning" />
        <Rule Id="CA2232" Action="Warning" />
        <Rule Id="CA2235" Action="Warning" />
        <Rule Id="CA2236" Action="Warning" />
        <Rule Id="CA2237" Action="Warning" />
        <Rule Id="CA2238" Action="Warning" />
        <Rule Id="CA2240" Action="Warning" />
        <Rule Id="CA2241" Action="Warning" />
        <Rule Id="CA2242" Action="Warning" />
        <Rule Id="CA1012" Action="Warning" />
    </Rules>
    <Rules AnalyzerId="StyleCop.Analyzers" RuleNamespace="StyleCop.Analyzers">
        <Rule Id="SA1305" Action="Warning" />
        <Rule Id="SA1412" Action="Warning" />
        <Rule Id="SA1600" Action="None" />
        <Rule Id="SA1609" Action="Warning" />
    </Rules>
</RuleSet>
```
    
Of course you can choose to ignore rules (Action="None") and get warnings (Action="Warning") or errors (Action="Error") in the case your code is not compliant.
      
## 6. Build the application and check the warnings
---
Run the following command:
          
``` powershell
dotnet build
```
   
You can get the code <a href="https://github.com/cmendible/dotnetcore.samples/tree/main/codeanalysis" target="_blank">here</a>.

Hope it helps!