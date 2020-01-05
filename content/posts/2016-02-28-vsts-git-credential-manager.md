---
author: Carlos Mendible
categories:
- devops
date: "2016-02-28T22:12:41Z"
description: VSTS Git Credential Manager
tags: ["Git", "VSTS"]
title: VSTS Git Credential Manager
---
Since the year started I've been working hard with **Visual Studio Team Services (VSTS)** with **Git** as source control. I was getting tired of entering my git credentials on each clone, pull or push, on Windows or Mac OS X,  so this weekend I decided to surf the web and look for a multi-platform solution.

I found exactly what I needed: **Git Credential Manager (GCM)** a Git credential helper that assists with multi-factor authentication and the best thing is that it works for [windows](https://github.com/Microsoft/Git-Credential-Manager-for-Windows), [mac or linux](https://github.com/Microsoft/Git-Credential-Manager-for-Mac-and-Linux/blob/master/Install.md).

To try it on my mac I performed the following steps:

  1. brew update
  2. brew install git-credential-manager
  3. git-credential-manager install

Then I tried to clone a repo hosted in VSTS and the GCM opened a web browser to let me authenticate to my account using OAuth 2.0 as shown in the following picture:

![git credential manager](/wp-content/uploads/2016/02/gcm.jpg)

Once authenticated, the clone command completed without issues and could pull or push without Git asking for credentials.

Tomorrow I'll try it on Windows!

Hope it helps!

**UPDATE:**

I did try it on Windows and everything worked like a charm!