---
author: Carlos Mendible
categories:
- devops
date: "2016-06-01T13:17:22Z"
description: Backup your Team Services Git Repositories with VSTS Vault
images: ["/wp-content/uploads/2016/06/vault.jpg"]
tags: ["backup", "git"]
title: Backup your Team Services Git Repositories with VSTS Vault
url: /2016/06/01/backup-team-services-git-repositories-vsts-vault/
---
**Backup your Team Services Git Repositories with VSTS Vault**: A simple windows service or console application designed to keep a local copy of all your code.

Since January 2016 we've moved the source code of more than 30 projects to Git repositories hosted by <a href="https://www.visualstudio.com/en-us/products/visual-studio-team-services-vs.aspx" target="_blank">Visual Studio Team Services</a>.

Moving to the cloud has some downsides and one thing that kept us thinking was the need to be sure that in the event of an internet connection failure or a Team Services outage we would be able to access our code and deploy hot fixes for our applications as part of our data protection plan.

Surfing on the web I found this post: <a href="http://blog.orbitone.com/post/Visual-Studio-Online-Backup-Tool" target="_blank">Free Visual studio online backup tool</a> that talks about <a href="https://github.com/OrbitOne/VSOBackup" target="_blank">VSOBackup</a> a free tool to create backups of your code by dates.

We needed a solution capable of minimizing the RPO (Recovery Point Objetive) and where the backups could be connected to an on-premises Git Server such as <a href="http://gitblit.com/" target="_blank">GitBlit</a>.

Those are the main reason we could not use <a href="https://github.com/OrbitOne/VSOBackup" target="_blank">VSOBackup</a> and why, inspired by such a great code base, I created a Windows service: <a href="https://github.com/cmendible/Vsts.Vault" target="_blank">VSTS.Vault</a> with the capability of cloning your Git repositories and then pulling for changes every 5 minutes which is the default setting.

Vsts.Vault is a Windows Service and to use it you'll need to follow these steps:

## Create Alternate Credentials for your Visual Studio Team Services Account
---
If you haven't setup alternate credentials for your <a href="https://www.visualstudio.com/en-us/products/visual-studio-team-services-vs.aspx" target="_blank">Visual Studio Team Services</a> Account follow the instructions of the Alternate Access Credentials section of the following doc: <a href="https://www.visualstudio.com/docs/report/analytics/client-authentication-options" target="_blank">Client Authentication Options</a>

## Configure VSTS.Vault Settings
---
Modify as needed the following parameters in the Vsts.Vault.exe.config file 
    
``` xml  
   <applicationSettings>
    <Vsts.Vault.WindowsService.Properties.Settings>
      <setting name="BackupInterval" serializeAs="String">
        <value>300000</value>
      </setting>
      <setting name="ServiceName" serializeAs="String">
        <value>Vsts.Vault</value>
      </setting>
    </Vsts.Vault.WindowsService.Properties.Settings>
  </applicationSettings>
 
  <VaultConfiguration
    Username="[User name as specified in your Team Services Alternate Authentication Credentials]"
    UserEmail="[An email required just required by Git pull through LibGit2Sharp]"
    Password="[Password as specified in your Team Services Alternate Authentication Credentials]"
    Account="[The name of your Team Services account (Not the full Url)]"
    TargetFolder="[Full path of the folder where the copy of your repositories will live]" />
```

## Install the VSTS.Vault service
---
Run the following command 
    
``` powershellinstallutil Vsts.Vault.exe
```

If you look through the [VSTS.Vault](https://github.com/cmendible/Vsts.Vault) code base you'll see some interesting features:

  1. Use of MEF as to implement dependency injection
  2. Use of [LibGit2Sharp](https://github.com/libgit2/libgit2sharp)
  3. Use of <a href="https://www.visualstudio.com/en-us/products/visual-studio-team-services-vs.aspx" target="_blank">Visual Studio Team Services</a> REST API
  4. And the use of the Retry Pattern

To finish this post I have to say that We've been running the service now for more than a month and connected the resulting repository backups to <a href="http://gitblit.com/" target="_blank">GitBlit</a> without issues.

Hope it helps!