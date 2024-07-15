---
author: Carlos Mendible
categories:
- azure
date: "2024-07-15T10:00:00Z"
description: 'Exploring AKS Automatic'
images: ["/assets/img/posts/aks.png"]
draft: false
tags: ["aks", "containers"]
title: 'Exploring AKS Automatic'
---

Azure Kubernetes Service (AKS) Automatic is a new SKU that simplifies the management of your AKS clusters. With this SKU, Azure ensures that your cluster is production ready with built-in best practice and a great code to kubernetes experience.

## Creating an AKS Automatic cluster

Creating an AKS cluster with the Automatic SKU is as simple as running the following Azure CLI command:

``` shell
az aks create \
    --resource-group <resource group name> \
    --name <cluster name> \
    --sku automatic \
    --generate-ssh-keys
```

Deploying with Bicep is also simple, but an agent pool profile is required:

``` bicep
@description('The name of the managed cluster resource.')
param clusterName string = 'aks-automatic'

@description('The location of the managed cluster resource.')
param location string = resourceGroup().location

resource aks 'Microsoft.ContainerService/managedClusters@2024-03-02-preview' = {
  name: clusterName
  location: location  
  sku: {
		name: 'Automatic'
  		tier: 'Standard'
  }
  properties: {
    agentPoolProfiles: [
      {
        name: 'systempool'
        count: 3
        vmSize: 'Standard_DS4_v2'
        osType: 'Linux'
        mode: 'System'
      }
    ]
  }
  identity: {
    type: 'SystemAssigned'
  }
}
```

> With just a few lines and almost no configuration you get a production-ready cluster.

## What is pre-configured (built-in) with AKS Automatic?

Once an AKS Automatic cluster is deployed it's full configuration can be retrieved using the Azure portal JSON view for the service. The JSON object is to large to be included here, but I'll highlight some of the most important parts of it.

### SKU

The SKU of the cluster is `Automatic` and the tier is `Standard`.

``` json
"sku": {
    "name": "Automatic",
    "tier": "Standard"
},
```

This means that the cluster is **SLA enabled** and also that it can be migrated to a Standard SKU if needed.

### Node Pools (Agent Pool Profile)

The cluster has a single agent pool named `systempool` with 3 nodes of size `Standard_DS4_v2` and running AzureLinux.

``` json
{
    "name": "systempool",
    "count": 3,
    "vmSize": "Standard_DS4_v2",
    "osDiskSizeGB": 128,
    "osDiskType": "Ephemeral",
    "kubeletDiskType": "OS",
    "maxPods": 250,
    "type": "VirtualMachineScaleSets",
    "availabilityZones": [
        "1",
        "2",
        "3"
    ],
    "enableAutoScaling": false,
    ...
    "orchestratorVersion": "1.28",
    "currentOrchestratorVersion": "1.28.10",
    "nodeTaints": [
        "CriticalAddonsOnly=true:NoSchedule"
    ],
    "mode": "System",
    "osType": "Linux",
    "osSKU": "AzureLinux",
    "nodeImageVersion": "AKSCBLMariner-V2gen2-202406.25.0",
    "upgradeSettings": {
        "maxSurge": "10%"
    },
    ...
}
```

The pool has Availability Zones enabled, auto-scaling disabled (more on this later) and the nodes are tainted with `CriticalAddonsOnly=true:NoSchedule` so no workloads are scheduled on them.

### Addons (Addon Profiles)

The follwoing addons are enabled by default in the cluster:

* **Azure Key Vault Secrets Provider**: allows for the integration of an Azure Key Vault as a secret store with an Azure Kubernetes Service (AKS) cluster via a CSI volume.
* **Azure Policy**: apply and enforce built-in security policies on your Azure Kubernetes Service (AKS) clusters using Azure Policy.
* **OMS Agent**: enables monitoring of the Azure Kubernetes Service (AKS) cluster.

``` json
"addonProfiles": {
    "azureKeyvaultSecretsProvider": {
        "enabled": true,
        "config": {
            "enableSecretRotation": "true"
        },
        "identity": {
            "resourceId": "...",
            ...
        }
    },
    "azurepolicy": {
        "enabled": true,
        "config": null,
        "identity": {
            "resourceId": "...",
            ...
        }
    },
    "omsAgent": {
        "enabled": true,
        "config": {
            "logAnalyticsWorkspaceResourceID": "...",
            "useAADAuth": "true"
        }
    }
},
```

> Note the use of Managed Identities and Microsoft Entra ID integration.

### Azure CNI Overlay and Cilium (Network Profile)

The network profile of the cluster is configured with Azure CNI, overlay mode, and Cilium network policy and data plane. This way only the Kubernetes cluster nodes are assigned IPs from subnets while Pods receive IPs from a private CIDR provided at the time of cluster creation.

With suppor for eBPF, Azure CNI Powered by Cilium provides the following benefits:

* Functionality equivalent to existing Azure CNI and Azure CNI Overlay plugins
* Improved Service routing
* More efficient network policy enforcement
* Better observability of cluster traffic
* Support for larger clusters (more nodes, pods, and services)

``` json
"networkProfile": {
    "networkPlugin": "azure",
    "networkPluginMode": "overlay",
    "networkPolicy": "cilium",
    "networkDataplane": "cilium",
    ...
}
```

### Microsoft Entra ID integration and cluster authentication (AAD Profile)

The cluster is configured with both Azure AD integration and Azure RBAC enabled.

``` json
"aadProfile": {
    "managed": true,
    "enableAzureRBAC": true,
    "tenantID": "<tenant id>"
},
```

Also the local accounts are disabled for the cluster, which means that you'll have to use `kubelogin` to access the cluster.

``` json
"disableLocalAccounts": true,
```

### Auto Upgrade Profile

The cluster is configured with auto-upgrade enabled and the upgrade channel set to `stable`.

``` json
"autoUpgradeProfile": {
    "upgradeChannel": "stable",
    "nodeOSUpgradeChannel": "NodeImage"
},
```

Auto-upgrade Stable channel automatically upgrades the cluster to the latest supported patch release on minor version N-1, where N is the latest supported minor version.

### Image Cleaner and Workload Identity (Security Profile)

The cluster is configured with the image cleaner enabled and workload identity enabled.

Image Cleaner is an AKS addon based on the OSS project `Eraser` used to remove unused images with vulnerabilities from the cluster nodes.

Workload Identity allows to easily cnfigure the use of Microsoft Entra application credentials or managed identities to access Microsoft Entra protected resources from within the cluster.

``` json
"securityProfile": {
    "imageCleaner": {
        "enabled": true,
        "intervalHours": 168
    },
    "workloadIdentity": {
        "enabled": true
    }
},
```

### Deployment safeguards (Safeguards Profile)

The cluster is configured with the Safeguards Profile enabled and set to `Warning`. Deployment safeguards programmatically assess your clusters at creation or update time for compliance. 

In warning mode, the deployment safeguards will not block the deployment of resources that are not compliant with the policy. Instead, it will dispaly warning messages in the code terminal to alert of any noncompliant cluster configurations.

``` json
"safeguardsProfile": {
    "level": "Warning",
    "version": "v1.0.0",
    "systemExcludedNamespaces": [
        "kube-system",
        "calico-system",
        "tigera-system",
        "gatekeeper-system"
    ]
},
```

### Ingress Controller (Ingress Profile)

The cluster is configured with the Web App Routing addon enabled. At the time of writing, this addon provides and easy configuration of managed NGINX Ingress controllers based on Kubernetes NGINX Ingress controller as well as integration with Azure DNS for public and private zone management and SSL termination with certificates stored in Azure Key Vault.

``` json
"ingressProfile": {
    "webAppRouting": {
        "enabled": true,
        "dnsZoneResourceIds": null,
        "identity": {
            "resourceId": "...",
            ...
        }
    }
},
```

### KEDA and VPA (Workload Auto Scaler Profile)

The cluster is configured with KEDA and Vertical Pod Autoscaler enabled.

``` json
"workloadAutoScalerProfile": {
    "keda": {
        "enabled": true
    },
    "verticalPodAutoscaler": {
        "enabled": true,
        "addonAutoscaling": "Unspecified"
    }
},
```

### Managed Prometheus and Grafana (Azure Monitor Profile)

The cluster is configured with Azure Monitor and Container Insights enabled. This configuration is optional but considered a best practice if your orggabization is not using a third-party monitoring solution.

``` json
"azureMonitorProfile": {
    "metrics": {
        "enabled": true,
        "kubeStateMetrics": {
            "metricLabelsAllowlist": "",
            "metricAnnotationsAllowList": ""
        }
    },
    "containerInsights": {
        "enabled": true,
        "logAnalyticsWorkspaceResourceId": "..."
    }
},
```

### Node Autoprovisioning (Node Provisioning Profile)

The cluster is configured with the node provisioning mode set to `Auto`. With this configuration, the cluster will automatically provision nodes as needed using NAP which is based on the Karpenter OSS project.

``` json
"nodeProvisioningProfile": {
    "mode": "Auto"
},
```

> Cluster autoscaler must be disabled when using node autoprovisioning.

Hope it helps!

**References:**

* [What is Azure Kubernetes Service (AKS) Automatic (preview)?](https://learn.microsoft.com/en-us/azure/aks/intro-aks-automatic)
* [Quickstart: Deploy an Azure Kubernetes Service (AKS) Automatic cluster (preview)](https://learn.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-automatic-deploy?pivots=bicep)
* [Use the Azure Key Vault provider for Secrets Store CSI Driver in an Azure Kubernetes Service (AKS) cluster](https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-driver)
* [Secure your Azure Kubernetes Service (AKS) clusters with Azure Policy](https://learn.microsoft.com/en-us/azure/aks/use-azure-policy)
* [Monitor Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/monitor-aks)
* [Overview of Overlay networking](https://learn.microsoft.com/en-us/azure/aks/azure-cni-overlay?tabs=kubectl#overview-of-overlay-networking)
* [Configure Azure CNI Powered by Cilium in Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/azure-cni-powered-by-cilium)
* [kubelogin](https://github.com/Azure/kubelogin)
* [Cluser Auto-upgrade channels](https://learn.microsoft.com/en-us/azure/aks/auto-upgrade-cluster?tabs=azure-cli#cluster-auto-upgrade-channels)
* [Use Microsoft Entra Workload ID with Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview?tabs=dotnet)
* [Image Cleaner](https://learn.microsoft.com/en-us/azure/aks/image-cleaner)
* [Use deployment safeguards to enforce best practices in Azure Kubernetes Service (AKS) (Preview)](https://learn.microsoft.com/en-us/azure/aks/deployment-safeguards)
* [Managed NGINX ingress with the application routing add-on](https://learn.microsoft.com/en-us/azure/aks/app-routing?tabs=default%2Cdeploy-app-default)
* [Node Autoprovisioning](https://learn.microsoft.com/en-us/azure/aks/node-autoprovision?tabs=azure-cli)