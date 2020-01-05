---
author: Carlos Mendible
categories:
- azure
- devops
crosspost_to_medium: true
date: "2017-08-02T16:57:08Z"
description: Create a Service Principal and write required parameters to a .azureauth
  file
image: /wp-content/uploads/2016/06/automation1.png
# tags: Automation AzureAD PowerShell Service Principal
title: Create a Service Principal and write required parameters to a .azureauth file
---
This week I had to repeat the process of creating a Service Principal in order to use the **Microsoft.Azure.Management.Fluent** lib with **.NET Core** so I decided it was time to script the process. With the following script you can **Create a Service Principal and write required parameters to a .azureauth file**.

You'll need the **AzureRM** PowerShell module installed:

``` powershell
Install-Module AzureRM 
```

Here is the code:

``` powershell
<#
    .SYNOPSIS
        New-ServicePrincipalAsReader is a PowerShell script to create a Read only Service Principal in Azure.
        The script will write a file ([subscriptionName].azureauth) with all the parameters needed to use the Microsoft.Azure.Management.Fluent lib.
        SECURITY: The file [subscriptionName].azureauth will contain the key for the  Service Principal.

    .DESCRIPTION
        New-ServicePrincipalAsReader is a PowerShell script to create a Read only Service Principal in Azure.
        The script will write a file ([subscriptionName].azureauth) with all the parameters needed to use the Microsoft.Azure.Management.Fluent lib.
        SECURITY: The file [subscriptionName].azureauth will contain the key for the  Service Principal.
        
    .PARAMETER subscriptionName
        The name of the subscription to connect.
    
    .PARAMETER servicePrincipalName
        The name of of the Service Principal. Default value is: logReader
    
    .NOTES 
        AUTHOR: Carlos Mendible 
        LASTEDIT: August 02, 2017 
#>
Param(
    [Parameter(Mandatory = $true)]
    [string]$subscriptionName,
    [Parameter(Mandatory = $false)]
    [string]$servicePrincipalName = "logReader"
)

# Creates an AesKey.
function Create-AesKey() {
    $aesManaged = New-Object "System.Security.Cryptography.AesManaged"
    $aesManaged.Mode = [System.Security.Cryptography.CipherMode]::CBC
    $aesManaged.Padding = [System.Security.Cryptography.PaddingMode]::Zeros
    $aesManaged.BlockSize = 128
    $aesManaged.KeySize = 256

    $aesManaged.GenerateKey()
    [System.Convert]::ToBase64String($aesManaged.Key)
}

# Create a Service Principal as subscription Reader.
# Resulting file is compatible with the Microsoft.Azure.Management.Fluent lib.
# SECURITY ALERT: Be careful with the file and its contents 
function New-ServicePrincipalAsReader($subscriptionName, $applicationName) {
    # Login to Azure
    Add-AzureRmAccount

    # Select the subscription
    Write-Host "Selecting subscription '$subscriptionName'";
    $subscriptionId = (Get-AzureRmSubscription -SubscriptionName $subscriptionName).Id
    Set-AzureRmContext -SubscriptionId $subscriptionId;

    # Get the Tenant Id
    $tenantId = (Get-AzureRmContext).Tenant.Id

    # Create an AD Application
    Write-Host "Creating the AD Application"
    $application = New-AzureRmADApplication `
        -DisplayName $applicationName `
        -HomePage "http://$applicationName" `
        -IdentifierUris "http://$applicationName"

    # Create the Key need to authenticate with this Application
    $keyValue = Create-AesKey
    $startDate = Get-Date
    $endDate = $startDate.AddYears(1)

    # Add a key to the Application
    Write-Host "Creating the AD Application Credential"
    New-AzureRmADAppCredential `
        -ApplicationId $application.ApplicationId `
        -Password $keyValue `
        -StartDate $startDate `
        -EndDate $endDate

    # Create the service principal
    Write-Host "Creating the Service Principal"
    $servicePrincipal = New-AzureRmADServicePrincipal -ApplicationId $application.ApplicationId
    Get-AzureRmADServicePrincipal -ObjectId $servicePrincipal.Id 

    # Make the service principal Reader
    Write-Host "Set the Principal as Reader"
    $ownerRole = $null
    $retries = 0;
    While ($ownerRole -eq $null -and $retries -le 6) {
        # Sleep here for a few seconds to allow the service principal application to become active 
        # (should only take a couple of seconds normally)
        Start-Sleep 15

        New-AzureRmRoleAssignment `
            -RoleDefinitionName Reader `
            -ServicePrincipalName $application.ApplicationId `
            -ErrorAction SilentlyContinue
        
        $ownerRole = Get-AzureRMRoleAssignment `
            -ServicePrincipalName $application.ApplicationId `
            -ErrorAction SilentlyContinue
    
        $retries++;
    }
    
    # Write the Authentication data to a file. Please be careful with where you save this file!!!
    $filePath = (Get-Location).Path + "\$servicePrincipalName.azureauth"
    Add-Content $filePath "subscription=$subscriptionId"
    Add-Content $filePath "client=$($application.ApplicationId)"
    Add-Content $filePath "tenant=$tenantId"
    Add-Content $filePath "key=$keyValue"
    Add-Content $filePath "managementURI=https\://management.core.windows.net/"
    Add-Content $filePath "baseURL=https\://management.azure.com/"
    Add-Content $filePath "authURL=https\://login.windows.net/"
    Add-Content $filePath "https\://graph.windows.net/"
}

New-ServicePrincipalAsReader `
    -subscriptionName $subscriptionName `
    -applicationName $servicePrincipalName
```

You can download the script from the <a href="https://gallery.technet.microsoft.com/Create-a-Service-Principal-179d0b36" target="_blank">Technet Script Gallery</a> or collaborate <a href="https://github.com/cmendible/azure.powershell.samples/tree/master/New-ServicePrincipalAsReader" target="_blank">here</a>.

Hope it helps!