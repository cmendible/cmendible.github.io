---
author: Carlos Mendible
categories:
- azure
- kubernetes
date: "2022-03-14T10:00:00Z"
description: 'AKS: Simulate Spot Node Eviction'
images: ["/assets/img/posts/aks.png"]
draft: true
tags: ["aks", "terraform", "spot"]
title: 'AKS: AKS: Simulate Spot Node Eviction'
---


## Use Terraform to create an AKS cluster with an extra node pool with Spot Virtual Machines.

### Create variables.tf with the following contents:

``` terraform	
variable "resource_group_name" {
  default = "aks-spot"
}

variable "location" {
  default = "West Europe"
}

variable "cluster_name" {
  default = "aks-spot"
}

variable "dns_prefix" {
  default = "aks-spot"
}
```

### Create providers.tf with the following contents:

``` terraform
terraform {
  required_version = "> 0.12"
  required_providers {
    azurerm = {
      source  = "azurerm"
      version = ">= 2.97.0"
    }
  }
}

provider "azurerm" {
  features {}
}
```

### Create main.tf with the following contents:

``` terraform	
# Create Resource Group
resource "azurerm_resource_group" "rg" {
  name     = var.resource_group_name
  location = var.location
}

# Create the VNET
resource "azurerm_virtual_network" "vnet" {
  name                = "aks-vnet"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  address_space       = ["10.0.0.0/16"]
}

# Create the subnet
resource "azurerm_subnet" "aks-subnet" {
  name                 = "aks-subnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.1.0/24"]
}

# Deploy Kubernetes
resource "azurerm_kubernetes_cluster" "k8s" {
  name                = var.cluster_name
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = var.dns_prefix

  default_node_pool {
    name                = "default"
    node_count          = 2
    vm_size             = "Standard_D2s_v3"
    os_disk_size_gb     = 30
    os_disk_type        = "Ephemeral"
    vnet_subnet_id      = azurerm_subnet.aks-subnet.id
    max_pods            = 15
    enable_auto_scaling = false
  }

  # Using Managed Identity
  identity {
    type = "SystemAssigned"
  }

  network_profile {
    # The --service-cidr is used to assign internal services in the AKS cluster an IP address. This IP address range should be an address space that isn't in use elsewhere in your network environment, including any on-premises network ranges if you connect, or plan to connect, your Azure virtual networks using Express Route or a Site-to-Site VPN connection.
    service_cidr = "172.0.0.0/16"
    # The --dns-service-ip address should be the .10 address of your service IP address range.
    dns_service_ip = "172.0.0.10"
    # The --docker-bridge-address lets the AKS nodes communicate with the underlying management platform. This IP address must not be within the virtual network IP address range of your cluster, and shouldn't overlap with other address ranges in use on your network.
    docker_bridge_cidr = "172.17.0.1/16"
    network_plugin     = "azure"
    network_policy     = "calico"
  }

  role_based_access_control {
    enabled = true
  }

  addon_profile {
    kube_dashboard {
      enabled = false
    }
  }
}

resource "azurerm_kubernetes_cluster_node_pool" "spot" {
  kubernetes_cluster_id = azurerm_kubernetes_cluster.k8s.id
  name                  = "spot"
  priority        = "Spot"
  eviction_policy = "Delete"
  spot_max_price  = -1 # note: this is the "maximum" price
  os_type = "Linux"
  vm_size             = "Standard_DS3_v2"
  os_disk_type        = "Ephemeral"
  node_count          = 1
  enable_auto_scaling = true
  max_count           = 3
  min_count           = 1
}

data "azurerm_resource_group" "node_resource_group" {
  name = azurerm_kubernetes_cluster.k8s.node_resource_group
}

# Assign the Contributor role to the AKS kubelet identity
resource "azurerm_role_assignment" "kubelet_contributor" {
  scope                = data.azurerm_resource_group.node_resource_group.id
  role_definition_name = "Contributor" #"Virtual Machine Contributor"?
  principal_id         = azurerm_kubernetes_cluster.k8s.kubelet_identity[0].object_id
}

resource "azurerm_role_assignment" "kubelet_network_contributor" {
  scope                = azurerm_virtual_network.vnet.id
  role_definition_name = "Network Contributor"
  principal_id         = azurerm_kubernetes_cluster.k8s.identity[0].principal_id
}
```

### Create outputs.tf with the following contents:

``` terraform 
output "node_resource_group" {
  value = data.azurerm_resource_group.node_resource_group.id
}
```

### Deploy the cluster:

Run the following commands:

``` shell
terraform init
terraform apply -auto-approve
```

### Simulate Spot Node Eviction:

Run the following commands:

``` powershell
$nodeResourceGroup=$(terraform output -raw node_resource_group)
$windowsScaleSet=$(az vmss list --resource-group $nodeResourceGroup --query "[].{name:name}[? contains(name,'spot')] | [0].name" --output tsv)
az vmss simulate-eviction --resource-group $nodeResourceGroup --name $windowsScaleSet --instance-id 0
```

### Query Log Analytics for Preempt status:

Run the following Log Analytics query:

``` shell
let endDateTime = now();
let startDateTime = ago(1h);
KubeNodeInventory
| where TimeGenerated < endDateTime
| where TimeGenerated >= startDateTime
| where Status contains "PreemptScheduled"
```

Hope it helps!!!