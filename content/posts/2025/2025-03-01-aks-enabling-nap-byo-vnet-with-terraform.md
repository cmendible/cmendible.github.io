---
author: Carlos Mendible
categories:
- azure
- kubernetes
date: "2025-03-01"
description: 'AKS BYO VNET: Enabling NAP with Terraform'
images: ["/assets/img/posts/aks.png"]
draft: false
tags: ["aks", "containers", "terraform"]
title: 'AKS BYO VNET: Enabling NAP with Terraform'
---

Last year I wrote a post about [Enabling NAP with Terraform](https://carlos.mendible.com/2024/12/09/aks-enabling-nap-with-terraform/).  While the post is still valid, I wanted to write about an scenario that many of you might be facing: Enabling NAP when bringing your own VNET.

So let's learn how to create an AKS cluster and enable Node Autoprovisioning (NAP) with Terraform when bringing your own VNET.

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
  default     = "aks-nap-byo-vnet"
}

variable "cluster_name" {
  description = "The name of the AKS cluster"
  type        = string
  default     = "aks-nap-byo-vnet"
}

resource "azurerm_resource_group" "rg" {
  name     = var.resource_group_name
  location = "Spain Central"
}

# Create VNET for AKS
resource "azurerm_virtual_network" "vnet" {
  name                = "private-network"
  address_space       = ["192.1.0.0/16"]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

# Create the Subnet for AKS nodes.
resource "azurerm_subnet" "aks_nodes" {
  name                 = "aks_nodes"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["192.1.0.0/16"]
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
    only_critical_addons_enabled = true
    vnet_subnet_id               = azurerm_subnet.aks_nodes.id # Use the subnet created above
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
    type = "UserAssigned"
    identity_ids = [
      azurerm_user_assigned_identity.mi.id # Use the user assigned managed identity
    ]
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

# Create a managed identity for the AKS cluster
resource "azurerm_user_assigned_identity" "mi" {
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  name                = "aks-nap"
}

# Assign the managed identity the required permissions to the AKS cluster
# Role assignments required by NAP so the managed identity can access the virtual network: Network Contributor
# https://github.com/Azure/karpenter-provider-azure/pull/326/files
resource "azurerm_role_assignment" "network_contributor" {
  scope                = azurerm_virtual_network.vnet.id
  role_definition_name = "Network Contributor"
  principal_id         = azurerm_user_assigned_identity.mi.principal_id
}
```

> NAP requieres that the managed identity has the `Network Contributor` role assigned to the VNET.

> WARNING: Do not enable cluster autoscaler when using NAP, as it will conflict with NAP.

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
* [AKS: Enabling  NAP with Terraform](https://carlos.mendible.com/2024/12/09/aks-enabling-nap-with-terraform/). 