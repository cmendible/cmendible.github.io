---
author: Carlos Mendible
categories:
- azure
- devops
crosspost_to_medium: true
date: "2019-05-16T15:00:00Z"
description: 'ARM: Enable Container Monitoring Solution on an existing Log Analytics
  Workspace'
image: /assets/img/posts/azure.png
published: true
tags: ["ARM", "Automation", "LogAnalytics"]
title: 'ARM: Enable Container Monitoring Solution on an existing Log Analytics Workspace'
---

Recently I had to update a bunch of [Log Analytics Workspaces](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&cad=rja&uact=8&ved=2ahUKEwiFrof58qDiAhXqyIUKHWPuBaIQFjAAegQIARAB&url=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fazure%2Fazure-monitor%2Flearn%2Fquick-create-workspace&usg=AOvVaw3DvKwidPs8__aX0fQ0vjQf) resources to enable the [Container Monitoring Solution](https://docs.microsoft.com/en-us/azure/azure-monitor/insights/containers). So I came up with this ARM Template that I want to share with you:

``` json
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "LogAnalyticsWorkspaceName": {
            "type": "string",
            "metadata": {
                "description": "Log Analytics Workspace name"
            }
        }
    },
    "variables": {
        "workspaceResourceId": "[resourceId('Microsoft.OperationalInsights/workspaces/', parameters('LogAnalyticsWorkspaceName'))]",
        "containerSolutionName": "[concat(parameters('LogAnalyticsWorkspaceName'), '-containers')]"
    },
    "resources": [
        {
            "type": "Microsoft.OperationsManagement/solutions",
            "apiVersion": "2015-11-01-preview",
            "name": "[variables('containerSolutionName')]",
            "location": "[resourceGroup().location]",
            "plan": {
                "name": "[variables('containerSolutionName')]",
                "product": "[concat('OMSGallery/', 'ContainerInsights')]",
                "promotionCode": "",
                "publisher": "Microsoft"
            },
            "properties": {
                "workspaceResourceId": "[variables('workspaceResourceId')]"
            }
        }
    ],
    "outputs": {}
}
```

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fcmendible%2Farmtemplates%2Fmaster%2Fcontainers%2Fenable-container-solution-existing-loganalytics-workspace%2Fazuredeploy.json" rel="nofollow">
    <img src="http://azuredeploy.net/deploybutton.png"/>
</a>

Hope it helps!
