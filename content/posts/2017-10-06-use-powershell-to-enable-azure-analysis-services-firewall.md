---
author: Carlos Mendible
categories:
- Azure
crosspost_to_medium: true
date: "2017-10-06T10:15:52Z"
description: Use PowerShell to Enable Azure Analysis Services Firewall
image: /wp-content/uploads/2017/06/powershell.png
# tags: AnalysisServices Firewall PowerShell
title: Use PowerShell to Enable Azure Analysis Services Firewall
---
Last week, firewall support was added to Azure Analysis Service (<a href="https://azure.microsoft.com/en-us/blog/azure-analysis-services-adds-firewall-support/" rel="noopener" target="_blank">https://azure.microsoft.com/en-us/blog/azure-analysis-services-adds-firewall-support/</a>). The thing is that, at the time of writing, there is no AzureRM cmdlet available to **Use PowerShell to Enable Azure Analysis Services Firewall**

So, with the help of **<a href="https://resources.azure.com" target="_blank">Resource Explorer</a>** I found which properties must be added to the service (resource) in order to enable the firewall:

  * ipV4FirewallSettings.firewallRules
  * ipV4FirewallSettings.enablePowerBIService

The following scripts use the **<a href="https://docs.microsoft.com/en-us/powershell/module/azurerm.resources/set-azurermresource?view=azurermps-4.1.0" target="_blank">Set-AzureRmResource</a>** cmdlet to set those properties.

``` powershell
Param 
(
    [Parameter(Mandatory = $true)][string]$resourceGroupName,
    [Parameter(Mandatory = $true)][string]$analysisServicesName,
    [Parameter(Mandatory = $true)][string]$firewallRuleName,
    [Parameter(Mandatory = $true)][string]$rangeStart,
    [Parameter(Mandatory = $true)][string]$rangeEnd)

function Add-AnalysisServicesFirewallRule($props, $firewallRuleName, $rangeStart, $rangeEnd) {
    # Check if the rule exists
    $existingRules = $props.ipV4FirewallSettings.firewallRules | `
        Where-Object {$_.firewallRuleName -eq $firewallRuleName}

    # Add the rule if needed
    if ($existingRules -eq $null) {
        # Create the rule object
        $ruleProperties = [ordered]@{
            "firewallRuleName" = $firewallRuleName;
            "rangeStart"       = $rangeStart;
            "rangeEnd"         = $rangeEnd;
        }
        $rule = New-Object -TypeName PSObject
        $rule | Add-Member -NotePropertyMembers $ruleProperties

        # Add the rule 
        $rules = [System.Collections.ArrayList]$props.ipV4FirewallSettings.firewallRules
        $index = $rules.Add($rule)
        $props.ipV4FirewallSettings.firewallRules = $rules
    }
}

function Add-FirewallSectionAndEnablePowerBIAccess($props) {
    # Check if ipV4FirewallSettings property exists
    $ipV4FirewallSettingsProperty = $props | Get-Member -Name "ipV4FirewallSettings"
    if (!$ipV4FirewallSettingsProperty) {
        # Create the ipV4FirewallSettings property and enable the PowerBI Service
        $props | Add-Member @{ipV4FirewallSettings = [ordered] @{ 
                "firewallRules"        = @()
                "enablePowerBIService" = $true
            }
        }
    }
}

# Get the AnalysisServices resource properties
$resource = Get-AzureRmResource `
    -ResourceGroupName $resourceGroupName `
    -ResourceType "Microsoft.AnalysisServices/servers" `
    -ResourceName $analysisServicesName

$props = $resource.Properties

Add-FirewallSectionAndEnablePowerBIAccess $props

Add-AnalysisServicesFirewallRule $props $firewallRuleName $rangeStart $rangeEnd

Set-AzureRmResource `
    -PropertyObject $props `
    -ResourceGroupName $resourceGroupName `
    -ResourceType "Microsoft.AnalysisServices/servers" `
    -ResourceName $analysisServicesName `
    -Force
```

If you save the script in a file (i.e Add-AzureRmAnalysisServicesFirewallRule.ps1) you can run it as follows:

``` powershell
.\Add-AzureRmAnalysisServicesFirewallRule.ps1
```

The script will ask you for:

  * The resource group name where the Analysis Service is deployed
  * The Analysis Service Name
  * The firewall rule name
  * The starting ip
  * The end ip

Get the code <a href="https://github.com/cmendible/azure.powershell.samples/blob/master/Add-AzureRmAnalysisServicesFirewallRule/Add-AzureRmAnalysisServicesFirewallRule.ps1" rel="noopener" target="_blank">here</a> or download the file from the <a href="https://gallery.technet.microsoft.com/Add-AzureRmAnalysisServices-6fd9d854" rel="noopener" target="_blank">Technet Gallery</a>.

Hope it helps!