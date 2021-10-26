---
author: Carlos Mendible
categories:
- azure
- devops
date: "2017-10-23T20:53:16Z"
description: Enable-AzureRmAnalysisServicesBackup is a small PowerShell script that
  uses the the Set-AzureRmResource cmdlet to enable backup location to an Azure Analysis
  Service instance.
images: ["/wp-content/uploads/2017/06/powershell.png"]
tags: ["AnalysisServices", "Automation", "BackUp", "PowerShell"]
title: Use PowerShell to Enable and Automate Azure Analysis Services Backup
---
In this post I'll show you how to **Use PowerShell to Enable and Automate Azure Analysis Backup**.

## **Enable Azure Analysis Service Backup**

Enable-AzureRmAnalysisServicesBackup is a small powershell script that uses the the Set-AzureRmResource cmdlet to enable backup location to an Azure Analysis Service instance.

Once you run it, the script will ask you for:

* The resource group name where the Analysis Service is deployed
* The name of the Azure Analysis Service instance.
* The name of te storage account where the backups will be placed
* The name of the container in the storage accoutn where the backups will be placed

``` powershell
Param
(
    [Parameter(Mandatory = $true)][string]$resourceGroupName,
    [Parameter(Mandatory = $true)][string]$analysisServicesName,
    [Parameter(Mandatory = $true)][string]$storageAccountName,
    [Parameter(Mandatory = $true)][string]$containerName)

function Add-AnalysisServicesBackupBlobContainer($props, $resourceGroupName, $storageAccountName, $containerName) {
    # Get storage account keys.
    $keys = Get-AzureRmStorageAccountKey `
        -StorageAccountName $storageAccountName `
        -ResourceGroupName $resourceGroupName

    # Use first key.
    $key = $keys[0].Value

    # Create an Azure Storage Context for the storage account.
    $context = New-AzureStorageContext -StorageAccountName $storageAccountName -StorageAccountKey $key

    # Create a 2 Years SAS Token
    $starTime = (Get-Date).AddMinutes(-5)
    $sasToken = New-AzureStorageContainerSASToken `
        -Name $containerName `
        -Context $context `
        -ExpiryTime $starTime.AddYears(2) `
        -Permission rwdlac `
        -Protocol HttpsOnly `
        -StartTime $starTime

    # Create the Container Uri
    $blobContainerUri = "https://$($storageAccountName).blob.core.windows.net/$($containerName)$($sasToken)"

    # Check the Analysis Server to see if the backupBlobContainerUri property exists.
    $backupBlobContainerUriProperty = $props | Get-Member -Name "backupBlobContainerUri"
    if (!$backupBlobContainerUriProperty) {
        # Add the property to the object.
        $props | Add-Member @{backupBlobContainerUri = ""}
    }

    # Set the container Uri
    $props.backupBlobContainerUri = $blobContainerUri
}

# Get the Analysis Services resource properties.
$resource = Get-AzureRmResource `
    -ResourceGroupName $resourceGroupName `
    -ResourceType "Microsoft.AnalysisServices/servers" `
    -ResourceName $analysisServicesName

$props = $resource.Properties

# Modify the backupBlobContainerUri.
Add-AnalysisServicesBackupBlobContainer $props $resourceGroupName $storageAccountName $containerName

# Save the properties.
Set-AzureRmResource `
    -PropertyObject $props `
    -ResourceGroupName $resourceGroupName `
    -ResourceType "Microsoft.AnalysisServices/servers" `
    -ResourceName $analysisServicesName `
    -Force
```

Get the code [here](https://github.com/cmendible/azure.powershell.samples/tree/main/Enable-AzureRmAnalysisServicesBackup) or download the file from the [Technet Gallery](https://gallery.technet.microsoft.com/Enable-AzureRmAnalysisServi-254a889c).

## **Automate Azure Analysis Service Backup**

Backup-AnalysisServices is a simple PowerShell workflow runbook that will help you automate the process of backing up an Azure Analysis Service Database.

The script receives 5 parameters:

* ResourceGroupName: The name of the resource group where the cluster resides
* AutomationCredentialName: The Automation credential holding username and password for Analysis Services
* AnalysisServiceDatabase: The Analysis Service Database
* AnalysisServiceServer: The Analysis Service Server (i.e. asazure://northeurope.asazure.windows.net/myanalysisservice)
* ConnectionName: The name of your automation connection account. Defaults to &#8216;AzureRunAsConnection'

``` powershell
<#
    .SYNOPSIS
        Backup-AnalysisServices is a simple PowerShell workflow runbook that will help you automate the process of backing up an Azure Analysis Service Database.

    .DESCRIPTION
        Backup-AnalysisServices is a simple PowerShell workflow runbook that will help you automate the process of backing up an Azure Analysis Service Database.

    .PARAMETER ResourceGroupName
        The name of the resource group where the cluster resides

    .PARAMETER AutomationCredentialName
        The Automation credential holding the username and password for Analysis Services

    .PARAMETER AnalysisServiceDatabase
        The Analysis Service Database

    .PARAMETER AnalysisServiceServer
        The Analysis Service Server

    .PARAMETER ConnectionName
        The name of your automation connection account. Defaults to  'AzureRunAsConnection'

    .NOTES
        AUTHOR: Carlos Mendible
        LASTEDIT: October 17, 2017
#>
workflow Backup-AnalysisServices {
    Param
    (
        [Parameter(Mandatory = $true)]
        [String]$ResourceGroupName,

        [Parameter(Mandatory = $true)]
        [String]$AutomationCredentialName,

        [Parameter(Mandatory = $true)]
        [String]$AnalysisServiceDatabase,

        [Parameter(Mandatory = $true)]
        [String]$AnalysisServiceServer,
        [Parameter(Mandatory = $false)]
        [String]$ConnectionName
    )

    # Requires the AzureRM.Profile and SqlServer PowerShell Modules

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

    # Get PSCredential
    $cred = Get-AutomationPSCredential -Name $AutomationCredentialName

    Write-Output "Starting Backup..."

    Backup-ASDatabase `
        –backupfile ("backup." + (Get-Date).ToString("yyMMdd") + ".abf") `
        –name $AnalysisServiceDatabase `
        -server $AnalysisServiceServer `
        -Credential $cred
}
```

Get the code [here](https://github.com/cmendible/azure.powershell.samples/tree/main/Backup-AnalysisServices) or download the file from the [Technet Gallery](https://gallery.technet.microsoft.com/Azure-Analysis-Backup-a06df3ad).

Hope it helps!