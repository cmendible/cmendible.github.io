---
author: Carlos Mendible
categories:
- azure
- devops
- dotnet
date: "2016-04-06T10:30:28Z"
description: Export Azure Resource Groups Templates
images: ["/wp-content/uploads/2016/04/ExportAzureRmResourceGroup.png"]
tags: ["ARM", "Templates", "Export", "PowerShell"]
title: Export Azure Resource Groups Templates
---
One of the great things about <a href="https://azure.microsoft.com/" target="_blank">Azure</a> is the possibility to **<a href="https://azure.microsoft.com/en-us/documentation/articles/resource-group-authoring-templates/" target="_blank">Export Azure Resource Groups Templates</a>**. Each template is a json file containing the exact configuration of the services you've provisioned in a <a href="https://azure.microsoft.com/en-us/documentation/articles/resource-group-overview/" target="_blank">Resource Group</a>.

Using this templates you can treat your <a href="https://en.wikipedia.org/wiki/Infrastructure_as_Code" target="_blank">Infrastructure as code</a> and repeatedly deploy your application during every stage of the application lifecycle in the same way, each and every time.

Navigate trough the following tabs to learn how to export your **<a href="https://azure.microsoft.com/en-us/documentation/articles/resource-group-authoring-templates/" target="_blank">Azure Resource Groups Templates</a>**

<div class="su-tabs su-tabs-style-default" data-active="1">
  <div class="su-tabs-nav">
    <span class="" data-url="" data-target="blank">Azure Portal</span><span class="" data-url="" data-target="blank">PowerShell</span><span class="" data-url="" data-target="blank">Azure CLI</span>

  
  <div class="su-tabs-panes">
    <div class="su-tabs-pane su-clearfix">
      <ol>
        <li>
          Loging to the <a href="http://portal.azure.com" target="_blank">Azure portal</a>
        </li>
        <li>
          Navigate to the <em>Resource Group</em> blade
        </li>
        <li>
          Click on <em>Settings</em>
        </li>
        <li>
          Once in the <em>Settings </em>blade click on <em>Export template</em>
        </li>
        <li>
          In the <em>Export resource group template</em> blade click <em>Save Template</em>
        </li>
      </ol>
      
  
<a href="http://carlos.mendible.com/wp-content/uploads/2016/04/ResourceGroup.png" rel="attachment wp-att-2791"><img class="alignleft size-medium wp-image-2791" src="http://carlos.mendible.com/wp-content/uploads/2016/04/ResourceGroup-300x67.png" alt="Resource Group Blade" width="300" height="67" srcset="/wp-content/uploads/2016/04/ResourceGroup-300x67.png 300w, /wp-content/uploads/2016/04/ResourceGroup-250x56.png 250w, /wp-content/uploads/2016/04/ResourceGroup.png 588w" sizes="(max-width: 300px) 100vw, 300px" /></a><a href="http://carlos.mendible.com/wp-content/uploads/2016/04/ResourceGroupSettings.png" rel="attachment wp-att-2771"><img class="alignleft size-medium wp-image-2771" src="http://carlos.mendible.com/wp-content/uploads/2016/04/ResourceGroupSettings-177x300.png" alt="Resource Group Settings Blade" width="177" height="300" srcset="/wp-content/uploads/2016/04/ResourceGroupSettings-177x300.png 177w, /wp-content/uploads/2016/04/ResourceGroupSettings-250x423.png 250w, /wp-content/uploads/2016/04/ResourceGroupSettings.png 314w" sizes="(max-width: 177px) 100vw, 177px" /></a> <a href="http://carlos.mendible.com/wp-content/uploads/2016/04/ExportResourceGroupTemplate.png" rel="attachment wp-att-2781"><img class="alignleft size-medium wp-image-2781" src="http://carlos.mendible.com/wp-content/uploads/2016/04/ExportResourceGroupTemplate-277x300.png" alt="Export Resource Group Template Blade" width="277" height="300" srcset="/wp-content/uploads/2016/04/ExportResourceGroupTemplate-277x300.png 277w, /wp-content/uploads/2016/04/ExportResourceGroupTemplate-250x270.png 250w, /wp-content/uploads/2016/04/ExportResourceGroupTemplate.png 579w" sizes="(max-width: 277px) 100vw, 277px" /></a>
        

With the latest release of <a href="http://resource group template" target="_blank">Azure Powershell</a> now it's possible to export your resource group template using the following command:
      
``` powershell
Export-AzureRmResourceGroup -ResourceGroupName  [-Path ]
```
      
Using <a href="https://github.com/azure/azure-xplat-cli" target="_blank">Microsoft Azure Cross Platform Command Line</a> and the following command you can save the template for the specified resource group.
          
``` powershell
azure group export [directory]
```
       
**04/15/2016 UPDATE:**
    
Now you can get the **.Net** code needed to automate your **Azure Resource Manager templates** deployments from code!!!
    
<a href="/wp-content/uploads/2016/04/resourcetemplate2dotnet.png"><img class="size-medium wp-image-3081 alignright" src="/wp-content/uploads/2016/04/resourcetemplate2dotnet-300x191.png" alt="Export Resource Template to .NET" width="300" height="191" srcset="/wp-content/uploads/2016/04/resourcetemplate2dotnet-300x191.png 300w, /wp-content/uploads/2016/04/resourcetemplate2dotnet-768x488.png 768w, /wp-content/uploads/2016/04/resourcetemplate2dotnet-250x159.png 250w, /wp-content/uploads/2016/04/resourcetemplate2dotnet.png 973w" sizes="(max-width: 300px) 100vw, 300px" /></a>
    
<ol>
  <li>
    Loging to the <a href="http://portal.azure.com" target="_blank">Azure portal</a>
  </li>
  <li>
    Navigate to the <em>Resource Group</em> blade
  </li>
  <li>
    Click on <em>Settings</em>
  </li>
  <li>
    Once in the <em>Settings </em>blade click on <em>Export template</em>
  </li>
  <li>
    In the <em>Export resource group template</em> blade click <em>.NET</em>
  </li>
</ol>     
    
Hope it helps!    