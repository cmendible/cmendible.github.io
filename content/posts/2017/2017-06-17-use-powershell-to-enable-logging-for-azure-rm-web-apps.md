---
author: Carlos Mendible
categories:
- azure
- devops
date: "2017-06-17T15:01:53Z"
description: Use PowerShell to Enable Logging for Azure RM Web Apps
images: ["/wp-content/uploads/2017/06/powershell.png"]
tags: ["PowerShell", "WebApp"]
title: Use PowerShell to Enable Logging for Azure RM Web Apps
---
Recently I client ask me to enable application and http logging for more than 20 AzureRM Web Apps. Furthermore the client wanted the logs to be stored in a Storage Account.

Issues started when I decided to **Use PowerShell to Enable Logging for Azure RM Web Apps**, because there is no equivalent to the ASM **<a href="https://docs.microsoft.com/en-us/powershell/module/azure/enable-azurewebsiteapplicationdiagnostic?view=azuresmps-4.0.0" target="_blank">Enable-AzureWebsiteApplicationDiagnostic</a>** cmdlet for ARM resources.

So how could I enable logging and the needed configuration? Well there is a handy cmdlet that can be used to set almost any property for an Azure ARM resource: **<a href="https://docs.microsoft.com/en-us/powershell/module/azurerm.resources/set-azurermresource?view=azurermps-4.1.0" target="_blank">Set-AzureRmResource</a>**

And how can anyone discover what properties must be set in order to enable or disable features? Well what I usually do is go to <a href="https://portal.azure.com" target="_blank">portal.azure.com</a> and configure a service as specified. Then I browse with **<a href="https://resources.azure.com" target="_blank">Resource Explorer</a>** to find the properties and values I need for the desired configuration.

In my clients case I needed to set the following properties:

  * applicationLogs.azureBlobStorage.level
  * applicationLogs.azureBlobStorage.sasUrl
  * applicationLogs.azureBlobStorage.retenionDays
  * httpLogs.azureBlobStorage.level
  * httpLogs.azureBlobStorage.sasUrl
  * httpLogs.azureBlobStorage.retenionDays
  * httpLogs.azureBlobStorage.enabled

So for the solution I just need to create a storage account and set those properties for every Web App found inside a Resource Group. This is the script I used:

``` powershell
function Add-IndexNumberToArray (
    [Parameter(Mandatory = $True)]
    [array]$array
) {
    for ($i = 0; $i -lt $array.Count; $i++) { 
        Add-Member -InputObject $array[$i] -Name "#" -Value ($i + 1) -MemberType NoteProperty 
    }
    $array
}

# Enable Web App Service Logs
function Enable-WebAppServiceLogs() {
    Add-AzureRmAccount

    [array]$SubscriptionArray = Add-IndexNumberToArray (Get-AzureRmSubscription) 
    [int]$SelectedSub = 0

    # use the current subscription if there is only one subscription available
    if ($SubscriptionArray.Count -eq 1) {
        $SelectedSub = 1
    }
    
    # Get SubscriptionID if one isn't provided
    while ($SelectedSub -gt $SubscriptionArray.Count -or $SelectedSub -lt 1) {
        Write-host "Please select a subscription from the list below"
        $SubscriptionArray | Select-Object "#", Id, Name | Format-Table
        try {
            $SelectedSub = Read-Host "Please enter a selection from 1 to $($SubscriptionArray.count)"
        }
        catch {
            Write-Warning -Message 'Invalid option, please try again.'
        }
    }

    # Select subscription
    Select-AzureRmSubscription -SubscriptionId $SubscriptionArray[$SelectedSub - 1].Id

    # Ask for the resource group.
    $resourceGroupName = Read-Host 'Resource Group name'

    # Ask for the storage account name
    $storageAccountName = Read-Host 'Storage account name'

    # Exit if resource group does not exists...
    $resourceGroup = Get-AzureRmResourceGroup -Name $resourceGroupName -ErrorAction SilentlyContinue
    if (!$resourceGroup) {
        Write-Verbose "$resourceGroup does not exists..."
        Exit 
    }

    # Create a new storage account.
    $storage = Get-AzureRmStorageAccount -ResourceGroupName $resourceGroupName -Name $storageAccountName -ErrorAction SilentlyContinue
    if (!$storage) {
        $storage = New-AzureRmStorageAccount -ResourceGroupName $resourceGroupName -Name $storageAccountName -Location $resourceGroup.Location -SkuName "Standard_ZRS"
    }

    # Create a container for the app logs 
    New-AzureStorageContainer -Context $storage.Context -Name "webapp-logs" -ErrorAction Ignore

    # Get a SAS token for the container
    $webSASToken = New-AzureStorageContainerSASToken -Context $storage.Context `
        -Name "webapp-logs" `
        -FullUri `
        -Permission rwdl `
        -StartTime (Get-Date).Date `
        -ExpiryTime (Get-Date).Date.AddYears(1)

    # Create a container for the http logs
    New-AzureStorageContainer -Context $storage.Context -Name "http-logs" -ErrorAction Ignore

    # Get a SAS token for the container
    $httpSASToken = New-AzureStorageContainerSASToken -Context $storage.Context `
        -Name "http-logs" `
        -FullUri `
        -Permission rwdl `
        -StartTime (Get-Date).Date `
        -ExpiryTime (Get-Date).Date.AddYears(1) 
    
    # Get all web app  in the resource group
    $resources = (Get-AzureRmResource).Where( {$_.ResourceType -eq "Microsoft.Web/sites" -and $_.ResourceGroupName -eq $resourceGroupName})

    # For each web app enable application and http logs 
    foreach ($resource in $resources) {
        # Property Object holding the log configuration.
        $propertiesObject = [ordered] @{
            'applicationLogs' = @{
                'azureBlobStorage' = @{
                    'level'           = 'Error' 
                    'sasUrl'          = [string]$webSASToken
                    'retentionInDays' = 30
                }
            }
            'httpLogs'        = @{
                'azureBlobStorage' = @{
                    'level'           = 'Error' 
                    'sasUrl'          = [string]$httpSASToken
                    'retentionInDays' = 30
                    'enabled'         = $true
                }
            }
        } 

        # Set the properties. Note that the resource type is: Microsoft.Web/sites/config and the resource name: [Web App Name]/logs
        Set-AzureRmResource `
            -PropertyObject $propertiesObject `
            -ResourceGroupName $resourceGroupName `
            -ResourceType Microsoft.Web/sites/config `
            -ResourceName "$($resource.ResourceName)/logs" -ApiVersion 2016-03-01 -Force          
    }
}

Enable-WebAppServiceLogs
```

If you save the script in a file (i.e Enable-WebAppServiceLogs.ps1) you can run it as follows:

``` powershell
.\Enable-WebAppServiceLogs.ps1
```

The script will ask you to:

  * login to your Azure account
  * select a subscription
  * write the name of the resource group containing the web apps
  * write the name of the storage account to hold the logs

Get the code here: <a href="https://github.com/cmendible/azure.powershell.samples/tree/main/Enable-WebAppServiceLogs"  target="_blank">https://github.com/cmendible/azure.powershell.samples/tree/main/Enable-WebAppServiceLogs</a>

Hope it helps!