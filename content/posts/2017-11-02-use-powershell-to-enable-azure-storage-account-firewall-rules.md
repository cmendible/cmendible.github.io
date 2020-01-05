---
author: Carlos Mendible
categories:
- azure
- devops
crosspost_to_medium: true
date: "2017-11-02T20:53:16Z"
description: Use PowerShell to enable Azure Storage Account Firewall Rules
image: /wp-content/uploads/2017/06/powershell.png
# tags: StorageAccount Automation Firewall PowerShell
title: Use PowerShell to enable Azure Storage Account Firewall Rules
---
In this post I'll show you how to **Use PowerShell to enable Azure Storage Account Firewall Rules**.

Be sure to be have [AzureRM PowerShell 4.4.1](https://www.powershellgallery.com/packages/AzureRM/4.4.1) module installed.

## **Login to your Azure Account**
Launch Powershell and start by Login to your Azure Account.

``` powershell
    Login-AzureRmAccount
```

## **Set Resource Group and Storage Account Name Variables**
Set the following variables

``` powershell
    $resourceGroupName = "[THE NAME OF THE RESOURCE GROUP]"
    $storageAccountName = "[THE NAME OF THE STORAGE ACCOUNT]"
```

## **Enable the Firewall**
To enable the firewall you'll need to **Deny** all trafic to the storage account using the **DefaultAction** parameter and then allow Azure Services to connect to it with the **Bypass parameter**.

``` powershell
    Update-AzureRmStorageAccountNetworkRuleSet -ResourceGroupName $resourceGroupName -Name $storageAccountName -DefaultAction Deny -Bypass AzureServices,Metrics,Logging
```

To allow an IP Address or an IP Address range you can run the following commands changing the value of the **IPAddressOrRange** parameter:

``` powershell
    Add-AzureRMStorageAccountNetworkRule -ResourceGroupName $resourceGroupName -AccountName $storageAccountName -IPAddressOrRange "79.159.46.90" 
```

``` powershell
    Add-AzureRMStorageAccountNetworkRule -ResourceGroupName $resourceGroupName -AccountName $storageAccountName -IPAddressOrRange "79.159.46.0/24" 
```
                
Hope it helps!