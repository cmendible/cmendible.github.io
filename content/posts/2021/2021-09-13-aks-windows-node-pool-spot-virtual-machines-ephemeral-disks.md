---
author: Carlos Mendible
categories:
- kubernetes
- azure
date: "2021-09-13T10:00:00Z"
description: 'AKS: Windows node pool with spot virtual machines and ephemeral disks'
images: ["/assets/img/posts/aks.png"]
draft: false
tags: ["windows", "ephemeral disks", "spot virtual machines"]
title: 'AKS: Windows node pool with spot virtual machines and ephemeral disks'
---

Some months ago a customer asked me if there was a way to deploy a Windows node pool with spot virtual machines and ephemeral disks in Azure Kubernetes Service (AKS).

The idea was to create a cluster that could be used to run Windows batch workloads and minimize costs by deploying the following:

* An AKS cluster with 2 linux nodes and ephemeral disks as the default node pool configuration.
* A Windows node pool with Spot Virtual Machines, ephemeral disks and auto-scaling enabled.
* Set the windows node pool minimum count and initial number of nodes set to 0.

To create a cluster with the desired configuration with terraform, follow the steps below:

## Define the terraform providers to use

Create a *providers.tf* file with the following contents:

``` yaml
terraform {
  required_version = "> 0.12"
  required_providers {
    azurerm = {
      source  = "azurerm"
      version = "~> 2.26"
    }
  }
}

provider "azurerm" {
  features {}
}
```

## Define the variables

Create a *variables.tf* file with the following contents:

``` yaml
variable "resource_group_name" {
  default = "aks-win"
}

variable "location" {
  default = "West Europe"
}

variable "cluster_name" {
  default = "aks-win"
}

variable "dns_prefix" {
  default = "aks-win"
}
```

## Define the resource group

Create a *main.tf* file with the following contents:

``` yaml
# Create Resource Group
resource "azurerm_resource_group" "rg" {
  name     = var.resource_group_name
  location = var.location
}
```

## Define the VNET for the cluster

Create a *vnet-server.tf* file with the following contents:

``` yaml
resource "azurerm_virtual_network" "vnet" {
  name                = "aks-vnet"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  address_space       = ["10.0.0.0/16"]
}

resource "azurerm_subnet" "aks-subnet" {
  name                 = "aks-subnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.1.0/24"]
}
```

## Define the AKS cluster

Create a *aks-server.tf* file with the following contents:

``` yaml
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

resource "azurerm_kubernetes_cluster_node_pool" "windows" {
  kubernetes_cluster_id = azurerm_kubernetes_cluster.k8s.id
  name                  = "win"
  priority        = "Spot"
  eviction_policy = "Delete"
  spot_max_price  = -1 # The VMs will not be evicted for pricing reasons.
  os_type = "Windows"
  # "The virtual machine size Standard_D2s_v3 has a cache size of 53687091200 bytes, but the OS disk requires 137438953472 bytes. Use a VM size with larger cache or disable ephemeral OS."
  # https://docs.microsoft.com/en-us/azure/virtual-machines/ephemeral-os-disks#size-requirements
  vm_size             = "Standard_DS3_v2"
  os_disk_type        = "Ephemeral"
  node_count          = 0
  enable_auto_scaling = true
  max_count           = 3
  min_count           = 0
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

## Deploy the AKS cluster

Run:

``` shell
terraform init
terraform apply
```

Get the credentials for the cluster:

``` shell
RESOURCE_GROUP="aks-win"
CLUSTER_NAME="aks-win"
az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME
```

To verify that there are no windows VMs running, execute:

``` shell
kubectl get nodes
```

you should see something like:

``` shell
NAME                              STATUS   ROLES   AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
aks-default-36675761-vmss000000   Ready    agent   80m   v1.20.9   10.0.1.4      <none>        Ubuntu 18.04.5 LTS   5.4.0-1056-azure   containerd://1.4.8+azure
aks-default-36675761-vmss000001   Ready    agent   80m   v1.20.9   10.0.1.20     <none>        Ubuntu 18.04.5 LTS   5.4.0-1056-azure   containerd://1.4.8+azure    
```

## Deploy a Windows workload:

To deploy a Windows workload, create a *windows_deployment.yaml* file with the following contents:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: servercore
  labels:
    app: servercore
spec:
  replicas: 1
  template:
    metadata:
      name: servercore
      labels:
        app: servercore
    spec:
      nodeSelector:
        "kubernetes.azure.com/scalesetpriority": "spot"
      containers:
      - name: servercore
        image: mcr.microsoft.com/dotnet/framework/samples:aspnetapp
        resources:
          limits:
            cpu: 1
            memory: 800M
          requests:
            cpu: .1
            memory: 150M
        ports:
          - containerPort: 80
      tolerations:
        - key: "kubernetes.azure.com/scalesetpriority"
          operator: "Equal"
          value: "spot"
          effect: "NoSchedule"
  selector:
    matchLabels:
      app: servercore
```

and deploy it to your cluster:

``` shell
kubectl apply -f windows_deployment.yaml
```

Note the following:

* The *kubernetes.azure.com/scalesetpriority* label is used to ensure that the workload is scheduled on a spot node.
* *tolerations* are used to ensure that the workload is scheduled on a spot node.
* Deployment will take a while (> 5 minutes) since the windows pool must scale up to fullfill the request.

Now check the nodes again:

``` shell
kubectl get nodes
```

this time you should see something like:

``` shell
NAME                              STATUS   ROLES   AGE    VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION     CONTAINER-RUNTIME
aks-default-36675761-vmss000000   Ready    agent   91m    v1.20.9   10.0.1.4      <none>        Ubuntu 18.04.5 LTS               5.4.0-1056-azure   containerd://1.4.8+azure
aks-default-36675761-vmss000001   Ready    agent   91m    v1.20.9   10.0.1.20     <none>        Ubuntu 18.04.5 LTS               5.4.0-1056-azure   containerd://1.4.8+azure
akswin000000                      Ready    agent   102s   v1.20.9   10.0.1.36     <none>        Windows Server 2019 Datacenter   10.0.17763.2114    docker://20.10.6   
```

If you check the pod events you'll find that the workload triggered a scale up:

``` shell
kubectl describe $(kubectl get po -l "app=servercore" -o name)   
```

I'll let you test what happens if you delete the deployment.

Hope it helps!!!

Please find the complete **terraform** configuration [here](https://github.com/cmendible/azure.samples/tree/main/aks_windows_spot_ephemeral_autoscaler/deploy)