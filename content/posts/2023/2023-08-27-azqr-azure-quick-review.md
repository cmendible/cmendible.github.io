---
author: Carlos Mendible
categories:
- azure
date: "2023-08-27T10:00:00Z"
description: "AZQR: Azure Quick Review"
images: ["/assets/img/posts/azure.png"]
published: true
tags: ["azure", "compliance", "assessment"]
title: "AZQR: Azure Quick Review"
---

## What is [Azure Quick Review](https://github.com/azure/azqr)?

If you are looking for a way to quickly assess the status and configuration of your Azure resources, you might want to try [Azure Quick Review (azqr)](https://github.com/azure/azqr): a command-line interface (CLI) tool that scans your Azure resources and generates an Excel report with detailed information and recommendations based on Azure's best practices.

[Azure Quick Review](https://github.com/azure/azqr) is an open source project hosted on GitHub. You can find the source code, documentation, installation instructions and usage examples [here](https://github.com/Azure/azqr)

## [Azure Quick Review (azqr)](https://github.com/azure/azqr) benefits

[Azure Quick Review](https://github.com/azure/azqr) can help you:

- Identify non-compliant or suboptimal configurations of your Azure resources.
- Get an overview of your Azure resource types, locations, SKUs, SLAs, Diagnostic Settings and more.
- Compare your resource naming conventions with the Cloud Adoption Framework (CAF) guidelines.
- Visualize and analyze your Azure resource data using Excel or Power BI Desktop (Windows only).

## [Azure Quick Review (azqr)](https://github.com/azure/azqr) installation

To use [Azure Quick Review](https://github.com/azure/azqr), you need to download it to your machine (Linux, Mac or Windows) or even inside Azure Cloud Shell:

### Linux and Azure Cloud Shell

``` bash
latest_azqr=$(curl -sL https://api.github.com/repos/Azure/azqr/releases/latest | jq -r ".tag_name" | cut -c1-)
wget https://github.com/Azure/azqr/releases/download/$latest_azqr/azqr-ubuntu-latest-amd64 -O azqr
chmod +x azqr
```

### Windows

``` bash
winget install azqr
```

### Mac

Download the latest release from [here](https://github.com/Azure/azqr/releases)

## Authentication and Permissions

### [Azure Quick Review](https://github.com/azure/azqr) supports the following authentication methods

- Azure CLI
- Service Principal. You'll need to set the following environment variables:
	- AZURE_CLIENT_ID
	- AZURE_CLIENT_SECRET
	- AZURE_TENANT_ID

> Although using Azure CLI is the easiest way to authenticate, it will slow down the scan process. If you want to scan a large number of resources, it's recommended to use a Service Principal instead.

### Permissions

[Azure Quick Review](https://github.com/azure/azqr) requires the following permissions:

- Subscription Reader

If you have any feedback, questions or issues, you can use the GitHub issues page to contact the developers and contributors: https://github.com/Azure/azqr/issues

Hope it helps!
