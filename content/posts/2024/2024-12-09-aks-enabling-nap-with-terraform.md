---
author: Carlos Mendible
categories:
- azure
- kubernetes
date: "2024-12-09"
description: 'AKS: Enabling NAP with Terraform'
images: ["/assets/img/posts/aks.png"]
draft: false
tags: ["aks", "containers", "terraform"]
title: 'AKS: Enabling NAP with Terraform'
---

Let's learn how to create an AKS cluster and enable Node Autoprovisioning (NAP) with Terraform. 

> Note: Since at the time of writing NAP is a preview feature, we will use the [azapi](https://registry.terraform.io/providers/azure/azapi/latest/docs) provider to enable it.

## Creating an AKS cluster and enable Node Autoprovisioning (NAP)

Create a file called `main.tf` with the following contents:

``` bash
terraform {
  required_providers {
    azapi = {
      source = "azure/azapi"
    }
    azurerm = {
      source = "hashicorp/azurerm"
    }
  }
}

provider "azapi" {
}

provider "azurerm" {
  features {}
}

variable "resource_group_name" {
  description = "The name of the resource group"
  type        = string
  default     = "aks-nap"
}

variable "cluster_name" {
  description = "The name of the AKS cluster"
  type        = string
  default     = "aks-nap"
}

resource "azurerm_resource_group" "rg" {
  name     = var.resource_group_name
  location = "Spain Central"
}

resource "azurerm_kubernetes_cluster" "k8s" {
  name                = var.cluster_name
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = var.cluster_name
  default_node_pool {
    name                         = "default"
    node_count                   = 1
    vm_size                      = "Standard_DS2_v2"
    only_critical_addons_enabled = true # tainting the nodes with CriticalAddonsOnly=true:NoSchedule to avoid scheduling workloads on the system node pool
    upgrade_settings {
      drain_timeout_in_minutes      = 0
      max_surge                     = "10%"
      node_soak_duration_in_minutes = 0
    }
  }
  network_profile {
    network_plugin      = "azure"
    network_plugin_mode = "overlay"
    network_policy      = "cilium"
    network_data_plane  = "cilium"
  }
  identity {
    type = "SystemAssigned"
  }
}

# Update the AKS cluster to enable NAP using the azapi provider
resource "azapi_update_resource" "nap" {
  type                    = "Microsoft.ContainerService/managedClusters@2024-09-02-preview"
  resource_id             = azurerm_kubernetes_cluster.k8s.id
  ignore_missing_property = true
  body = {
    properties = {
      nodeProvisioningProfile = {
        mode = "Auto"
      }
    }
  }
}
```

> Note: We set the `only_critical_addons_enabled` to true, to taint the nodes and avoid scheduling workloads on the system node pool.

Run the following commands to create the AKS cluster and enable NAP:

``` bash
export ARM_SUBSCRIPTION_ID=<your-subscription-id>
terraform init
terraform apply
```

## Deploy a NodePool CRD to the cluster

Create a file called `nodepool.yaml` with the following contents:

``` bash
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: Never
  template:
    spec:
      nodeClassRef:
        name: default
      requirements:
      - key: kubernetes.io/arch
        operator: In
        values:
        - amd64
      - key: kubernetes.io/os
        operator: In
        values:
        - linux
      - key: karpenter.sh/capacity-type
        operator: In
        values:
        - on-demand
      - key: karpenter.azure.com/sku-family
        operator: In
        values:
        - D
```

Run the following commands to deploy the NodePool CRD to the cluster:

``` bash
az aks get-credentials --resource-group aks-nap --name aks-nap
kubectl apply -f nodepool.yaml
```

## Deploy a pod to the cluster

``` bash
kubectl run busybox --image=busybox -- sh -c "sleep 3600"
```

After a while you should see a new node being provisioned to the cluster and the pod running on it.

Hope it helps!

**References:**

* [Node Autoprovisioning](https://learn.microsoft.com/en-us/azure/aks/node-autoprovision?tabs=azure-cli)