---
author: Carlos Mendible
categories:
- azure
- kubernetes
date: "2026-02-16"
description: 'AKS: Azure Container Storage with local NVMe using Terraform'
images: ["/assets/img/posts/aks.png"]
draft: false
tags: ["aks", "containers", "terraform", "azure-container-storage", "nvme"]
title: 'AKS: Azure Container Storage with local NVMe using Terraform'
---

Let's learn how to deploy an Azure Kubernetes Service (AKS) cluster with Azure Container Storage enabled using Terraform, leveraging local NVMe disks for high-performance storage.

> Azure Container Storage is a cloud-based volume management, deployment, and orchestration service built natively for containers. It integrates with Kubernetes so you can dynamically provision persistent volumes for stateful applications.

## When to use Azure Container Storage with NVMe

Azure Container Storage with local NVMe disks is the best option for Kubernetes workloads that need extremely high performance. Here are some scenarios where it can help:

- **High-speed caching layers**: Datasets and checkpoints for AI training, or model files used for AI inference.
- **High-performance databases**: Self-hosted databases like PostgreSQL, Cassandra, or Redis that include built-in replication and benefit from sub-millisecond latency.
- **Data-intensive analytics**: Processing pipelines that require fast, temporary storage for intermediate data.
- **Batch processing**: Temporary scratch space for compute-intensive batch jobs.
- **AI/ML workloads**: Frameworks like Ray and Kubeflow that require high IOPS and throughput.

> Note: Local NVMe disks are ephemeral. Data stored on these disks is lost if the VM is deallocated or redeployed. Ensure your application can tolerate data loss or has built-in replication.

## Prerequisites

Register the required resource provider in your Azure subscription:

``` bash
az provider register --namespace Microsoft.KubernetesConfiguration --wait
```

Verify the registration status:

``` bash
az provider show --namespace Microsoft.KubernetesConfiguration --query "registrationState"
```

## Creating an AKS cluster with Azure Container Storage

### 1. Create the providers.tf file

Create a file called `providers.tf` with the following contents:

``` terraform
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "4.60.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "3.0.1"
    }
  }
}

terraform {
  required_version = ">= 1.14.5"
}

provider "azurerm" {
  features {}
}

provider "kubernetes" {
  host                   = azurerm_kubernetes_cluster.k8s.kube_config.0.host
  client_certificate     = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.client_certificate)
  client_key             = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.client_key)
  cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.cluster_ca_certificate)
}
```

### 2. Create the variables.tf file

Create a file called `variables.tf` with the following contents:

``` terraform
variable "resource_group_name" {
  default = "rg-aks-container-storage"
}

variable "location" {
  default = "swedencentral"
}

variable "cluster_name" {
  default = "aks-container-storage"
}

variable "dns_prefix" {
  default = "aks-container-storage"
}
```

### 3. Create the main.tf file

Create a file called `main.tf` with the following contents:

``` terraform
data "azurerm_subscription" "current" {}

data "azurerm_client_config" "current" {}

# Create Resource Group
resource "azurerm_resource_group" "rg" {
  name     = var.resource_group_name
  location = var.location
}
```

### 4. Create the vnet.tf file

Create a file called `vnet.tf` with the following contents:

``` terraform
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

### 5. Create the aks.tf file

Create a file called `aks.tf` with the following contents:

``` terraform
# Deploy Kubernetes
resource "azurerm_kubernetes_cluster" "k8s" {
  name                              = var.cluster_name
  location                          = azurerm_resource_group.rg.location
  resource_group_name               = azurerm_resource_group.rg.name
  dns_prefix                        = var.dns_prefix
  oidc_issuer_enabled               = true
  workload_identity_enabled         = true
  role_based_access_control_enabled = true
  sku_tier                          = "Standard"

  default_node_pool {
    name                 = "default"
    node_count           = 3
    vm_size              = "Standard_L8s_v3" # Local NVMe disks require storage-optimized VMs
    os_disk_size_gb      = 30
    os_disk_type         = "Ephemeral"
    vnet_subnet_id       = azurerm_subnet.aks-subnet.id
    max_pods             = 30
    min_count            = 3
    max_count            = 6
    auto_scaling_enabled = true
    upgrade_settings {
      drain_timeout_in_minutes      = 0
      max_surge                     = "10%"
      node_soak_duration_in_minutes = 0
    }
  }

  # Using Managed Identity
  identity {
    type = "SystemAssigned"
  }

  network_profile {
    service_cidr        = "172.0.0.0/16"
    dns_service_ip      = "172.0.0.10"
    network_plugin      = "azure"
    network_plugin_mode = "overlay"
    network_data_plane  = "cilium"
    network_policy      = "cilium"
  }
}

// Deploy Azure Container Storage extension for AKS
resource "azurerm_kubernetes_cluster_extension" "container_storage" {
  name           = "acstor" # NOTE: the name parameter must be "acstor" for Azure CLI compatibility
  cluster_id     = azurerm_kubernetes_cluster.k8s.id
  extension_type = "microsoft.azurecontainerstoragev2"
}

// Create a StorageClass for Azure Container Storage
resource "kubernetes_storage_class_v1" "local" {
  metadata {
    name = "local"
  }

  depends_on = [azurerm_kubernetes_cluster_extension.container_storage]

  storage_provisioner    = "localdisk.csi.acstor.io"
  reclaim_policy         = "Delete"
  volume_binding_mode    = "WaitForFirstConsumer"
  allow_volume_expansion = true
}
```

> Note: We use `Standard_L8s_v3` VMs because local NVMe disks are only available on storage-optimized VMs or GPU accelerated VMs. The Azure Container Storage extension (`microsoft.azurecontainerstoragev2`) is deployed with the name `acstor` for Azure CLI compatibility.

### 6. Create the pod.tf file

Create a file called `pod.tf` to deploy a test pod that uses the Azure Container Storage:

``` terraform
resource "kubernetes_pod_v1" "fiopod" {
  metadata {
    name = "fiopod"
  }

  depends_on = [kubernetes_storage_class_v1.local]

  spec {
    node_selector = {
      "kubernetes.io/os" = "linux"
    }

    container {
      name  = "fio"
      image = "mayadata/fio"
      args  = ["sleep", "1000000"]
      volume_mount {
        name       = "ephemeralvolume"
        mount_path = "/volume"
      }
    }

    volume {
      name = "ephemeralvolume"
      ephemeral {
        volume_claim_template {
          spec {
            volume_mode        = "Filesystem"
            access_modes       = ["ReadWriteOnce"]
            storage_class_name = "local" # This should match the name of the StorageClass
            resources {
              requests = {
                storage = "10Gi"
              }
            }
          }
        }
      }
    }
  }
}
```

### 7. Create the outputs.tf file

Create a file called `outputs.tf` with the following contents:

``` terraform
output "cluster_name" {
  description = "The name of the AKS cluster"
  value       = azurerm_kubernetes_cluster.k8s.name
}

output "resource_group_name" {
  description = "The name of the resource group"
  value       = azurerm_resource_group.rg.name
}

output "cluster_id" {
  description = "The ID of the AKS cluster"
  value       = azurerm_kubernetes_cluster.k8s.id
}

output "cluster_location" {
  description = "The location of the AKS cluster"
  value       = azurerm_kubernetes_cluster.k8s.location
}
```

## Deploy the infrastructure

Run the following commands to deploy the AKS cluster with Azure Container Storage:

``` bash
export ARM_SUBSCRIPTION_ID=<your-subscription-id>
terraform init
terraform apply
```

## Verify the deployment

Get the AKS cluster credentials:

``` bash
az aks get-credentials --resource-group $(terraform output -raw resource_group_name) --name $(terraform output -raw cluster_name)
```

Verify the storage class was created:

``` bash
kubectl get storageclass local
```

Check the fiopod status:

``` bash
kubectl get pod fiopod
kubectl describe pod fiopod
```

## Run a benchmark (optional)

Once the pod is running, you can run a fio benchmark to test the NVMe performance:

``` bash
kubectl exec -it fiopod -- fio --name=random-write --ioengine=posixaio --rw=randwrite --bs=4k --size=1g --numjobs=1 --iodepth=1 --runtime=60 --time_based --directory=/volume
```

Hope it helps!

**References:**

* [What is Azure Container Storage?](https://learn.microsoft.com/azure/storage/container-storage/container-storage-introduction)
* [Use Azure Container Storage with local NVMe](https://learn.microsoft.com/azure/storage/container-storage/use-container-storage-with-local-disk)
* [Best practices for ephemeral NVMe data disks in AKS](https://learn.microsoft.com/azure/aks/best-practices-storage-nvme)
* [Storage options for applications in AKS](https://learn.microsoft.com/azure/aks/concepts-storage)
* [Sample code on GitHub](https://github.com/cmendible/azure.samples/tree/main/aks_azure_container_storage)
