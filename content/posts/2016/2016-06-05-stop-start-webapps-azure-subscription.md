---
author: Carlos Mendible
categories:
- azure
- devops
date: "2016-06-05T19:08:30Z"
description: Stop or Start all WebApps in your Azure subscription
images: ["/wp-content/uploads/2016/06/automation1.png"]
tags: ["Automation", "PowerShell", "WebApp", "WebSite"]
title: Stop or Start all WebApps in your Azure subscription
url: /2016/06/05/stop-start-webapps-azure-subscription/
---
**Stop or Start all WebApps in your Azure subscription** with [Stop-Start-All-WebApps.ps1](https://gallery.technet.microsoft.com/scriptcenter/Stop-or-Start-all-WebApps-9533dda6/file/153545/1/Stop-Start-All-WebApps.ps1) a simple PowerShell workflow runbook that will help you automate the process of stopping or starting every single WebApp (Website) you've deployed.

The script receives two required parameters:

  * **Stop**: If set to true stop the WebApps (Websites) otherwise start the WebApps (Websites)
  * **CredentialAssetName**: The name of a valid Credential Asset (AutomationPSCredential) you must have previously configured.

As you can see below, the script is quite simple and uses the following PowerShell Activities and Cmdlets:

<table>
    <tr>
      <td>
        Type
      </td>
      <td>
        Name
      </td>
      <td>
        Description
      </td>
    </tr>
    <tr>
      <td>
        Activity
      </td>
      <td>
        <a href="https://github.com/Azure/azure-content/blob/master/articles/automation/automation-credentials.md" target="_blank">Get-AutomationPSCredential</a>
      </td>
      <td>
        Gets a credential to use in a runbook or DSC configuration. Returns a <a href="http://msdn.microsoft.com/library/system.management.automation.pscredential" target="_blank">System.Management.Automation.PSCredential</a> object.
      </td>
    </tr>
    <tr>
      <td>
        Cmdlet
      </td>
      <td>
        <a href="https://msdn.microsoft.com/en-us/library/dn790372.aspx" target="_blank">Add-AzureAccount</a>
      </td>
      <td>
        Adds an authenticated account to use for Resource Manager cmdlet requests.
      </td>
    </tr>
    <tr>
      <td>
        Cmdlet
      </td>
      <td>
        <a href="https://msdn.microsoft.com/en-us/library/mt619267.aspx" target="_blank">Add-AzureRmAccount</a>
      </td>
      <td>
        Adds the Azure account to Windows PowerShell
      </td>
    </tr>
    <tr>
      <td>
        Cmdlet
      </td>
      <td>
        <a href="https://msdn.microsoft.com/en-us/library/azure/dn495127.aspx" target="_blank">Get-AzureWebsite</a>
      </td>
      <td>
        Gets Azure websites in the current subscription
      </td>
    </tr>
    <tr>
      <td>
        Cmdlet
      </td>
      <td>
        <a href="https://msdn.microsoft.com/en-us/library/azure/dn495185.aspx" target="_blank">Stop-AzureWebsite</a>
      </td>
      <td>
        Stops the specified webiste
      </td>
    </tr>
    <tr>
      <td>
        Cmdlet
      </td>
     <td>
        <a href="https://msdn.microsoft.com/en-us/library/azure/dn495288.aspx" target="_blank">Start-AzureWebsite</a>
      </td>
      <td>
        Starts the specified webiste
      </td>
    </tr>
</table>

Here is the script:

``` powershell
<# 
    .SYNOPSIS  
        Start or Stop all WebApps (Websites) in your subscription. 
 
    .DESCRIPTION 
        Runbook which allows you to start/stop all WebApps (Websites) in your subscription. 
 
    .PARAMETER Stop 
        If set to true: stop the WebApps (Websites). Otherwise start the WebApps (Websites) 
 
    .PARAMETER CredentialAssetName 
        The name of a working AutomationPSCredential 
         
    .NOTES 
        AUTHOR: Carlos Mendible 
        LASTEDIT: June 2, 2016 
#> 
Workflow Stop-Start-All-WebApps  
{ 
    # Parameters 
    Param( 
        [Parameter (Mandatory= $true)] 
        [bool]$Stop, 
         
        [Parameter (Mandatory= $true)] 
        [string]$CredentialAssetName 
       )   
        
    #The name of the Automation Credential Asset this runbook will use to authenticate to Azure. 
    $CredentialAssetName = $CredentialAssetName; 
     
    #Get the credential with the above name from the Automation Asset store 
    $Cred = Get-AutomationPSCredential -Name $CredentialAssetName 
    if(!$Cred) { 
        Throw "Could not find an Automation Credential Asset named '${CredentialAssetName}'. Make sure you have created one in this Automation Account." 
    } 
 
    #Connect to your Azure Account        
    Add-AzureRmAccount -Credential $Cred 
    Add-AzureAccount -Credential $Cred 
     
    $status = 'Stopped' 
    if ($Stop) 
    { 
        $status = 'Running' 
    } 
 
    # Get Running WebApps (Websites) 
    $websites = Get-AzureWebsite | where-object -FilterScript{$_.state -eq $status } 
     
    foreach -parallel ($website In $websites) 
    { 
        if ($Stop) 
        { 
            $result = Stop-AzureWebsite $website.Name 
            if($result) 
            { 
                Write-Output "- $($website.Name) did not shutdown successfully" 
            } 
            else 
            { 
                Write-Output "+ $($website.Name) shutdown successfully" 
            } 
        } 
        else 
        { 
            $result = Start-AzureWebsite $website.Name 
            if($result) 
            { 
                Write-Output "- $($website.Name) did not start successfully" 
            } 
            else 
            { 
                Write-Output "+ $($website.Name) started successfully" 
            } 
        }  
    }     
}
```

You can download the script from the <a href="https://gallery.technet.microsoft.com/scriptcenter/Stop-or-Start-all-WebApps-9533dda6" target="_blank">Technet Script Gallery</a>, import it as a runbook for your automation account in the Azure portal or simply copy the code from this post.

Hope it helps!