---
author: Carlos Mendible
categories:
- devops
date: "2017-11-14T02:15:00Z"
description: Use PowerShell to create a Microsoft Teams Channel
images: ["/wp-content/uploads/2017/06/powershell.png"]
tags: ["PowerShell", "MicrosoftTeams"]
title: Use PowerShell to create a Microsoft Teams Channel
---
For those of you who have been trying to automate anything related to Microsoft Teams, let me tell you that there is a new PowerShell Module in town: [Microsoft Teams 0.9.0](https://www.powershellgallery.com/packages/MicrosoftTeams/0.9.0) which you can install with the following command:

``` powershell
    Install-Module MicrosoftTeams
```

Now to automate the channel creation In A Team you can simply:

## 1. Create a file createTeamChannel.ps1 with the following contents:

``` powershell
    Param
    (
        [Parameter(Mandatory = $true)][string]$username,
        [Parameter(Mandatory = $true)][securestring]$password,
        [Parameter(Mandatory = $true)][string]$tenantId,
        [Parameter(Mandatory = $true)][string]$teamName,
        [Parameter(Mandatory = $true)][string]$channelName)

    # Create PSCredential
    $credential = New-Object System.Management.Automation.PSCredential($username, $password)

    # Connect to Microsoft Teams
    Connect-MicrosoftTeams -TenantId $tenantId -Credential $credential

    # Get the Team
    $team = Get-Team | Where-Object { $_.DisplayName -eq $teamName}

    if ($team) {
        # Create the new Team Channel
        New-TeamChannel -GroupId $team.GroupId -DisplayName $channelName -Description $channelName
    }
    else {
        throw "Team: $teamName does not exist."
    }
```

## 2. Run the following command in PowerShell

``` powershell
    $password = ConvertTo-SecureString "YOUR PASSWORD" -AsPlainText -Force
    .\createTeamChannel.ps1 username@tenant.com $password $tenantId MonitoringIssues Incident101
```

Hope it helps!