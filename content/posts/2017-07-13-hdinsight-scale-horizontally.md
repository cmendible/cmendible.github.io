---
author: Carlos Mendible
categories:
- azure
- devops
crosspost_to_medium: true
date: "2017-07-13T11:07:13Z"
description: 'HDInsight: Scale Horizontally'
image: /wp-content/uploads/2016/06/automation1.png
tags: ["Automation", "HDInsight", "PowerShell"]
title: 'HDInsight: Scale Horizontally'
---
**HDInsight: Scale Horizontally** with [Scale-HDInsightClusterNodes.ps1](https://gallery.technet.microsoft.com/Scale-your-HDInsight-f57bb4d8) a PowerShell workflow that will help you automate the process of scaling your cluster.

The script receives 4 parameters:

  * **ResourceGroupName**: The name of the resource group where the cluster resides
  * **ClusterName**: The name of your HDInsight cluster
  * **Nodes**: The number of nodes you want for the cluster
  * **ConnectionName**: The name of your automation connection account

and requires the following PowerShell modules: **AzureRM.Profile**, **AzureRM.HDInsight**

Here is the code:

``` powershell
<#
    .SYNOPSIS
        Scale-HDInsightClusterNodes is a simple PowerShell workflow runbook that will help you automate the process of scaling in or out your HDInsight clusters depending on your needs.
    
    .DESCRIPTION
        Scale-HDInsightClusterNodes is a simple PowerShell workflow runbook that will help you automate the process of scaling in or out your HDInsight clusters depending on your needs.

    .PARAMETER ResourceGroupName
        The name of the resource group where the cluster resides
    
    .PARAMETER ClusterName
        The name of your HDInsight cluster
    
    .PARAMETER Nodes
        The number of nodes you want for the cluster

    .PARAMETER ConnectionName
        The name of your automation connection account
   
    .NOTES 
        AUTHOR: Carlos Mendible 
        LASTEDIT: June 13, 2017 
#>
Workflow Scale-HDInsightClusterNodes {
    Param
    (   
        [Parameter(Mandatory = $true)]
        [String]$ResourceGroupName,

        [Parameter(Mandatory = $true)]
        [String]$ClusterName,

        [Parameter(Mandatory = $true)]
        [Int]$Nodes,

        [Parameter(Mandatory = $false)]
        [String]$ConnectionName
    )

    # This Workflow requieres the following PowerShell modules: AzureRM.Profile, AzureRM.HDInsight

    $automationConnectionName = $ConnectionName
    if (!$ConnectionName) {
        $automationConnectionName = "AzureRunAsConnection"
    }
	
    # Get the connection by name (i.e. AzureRunAsConnection)
    $servicePrincipalConnection = Get-AutomationConnection -Name $automationConnectionName         

    Write-Output "Logging in to Azure..."
    
    Add-AzureRmAccount `
        -ServicePrincipal `
        -TenantId $servicePrincipalConnection.TenantId `
        -ApplicationId $servicePrincipalConnection.ApplicationId `
        -CertificateThumbprint $servicePrincipalConnection.CertificateThumbprint
 
    Write-Output "Scaling cluster $ClusterName to $Nodes nodes..."
    Set-AzureRmHDInsightClusterSize `
        -ResourceGroupName $ResourceGroupName `
        -ClusterName $ClusterName `
        -TargetInstanceCount $Nodes
}
```

You can download the script from the <a href="https://gallery.technet.microsoft.com/Scale-your-HDInsight-f57bb4d8" target="_blank">Technet Script Gallery</a>, import it as a runbook for your automation account in the Azure portal or simply copy the code from this post.

Hope it helps!