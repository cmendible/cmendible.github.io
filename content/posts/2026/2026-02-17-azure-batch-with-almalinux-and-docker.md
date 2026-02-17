---
author: Carlos Mendible
categories:
- azure
date: "2026-02-17"
description: 'Deploy an Azure Batch pool running AlmaLinux 9 with Docker CE using Terraform'
images: ["/assets/img/posts/azure.png"]
draft: false
tags: ["azure-batch", "almalinux", "docker", "terraform"]
title: 'Azure Batch with AlmaLinux and Docker'
---

In this post, I'll show you how to deploy an Azure Batch pool running AlmaLinux 9 with Docker CE using Terraform. This is particularly useful when you need to run containerized workloads on Azure Batch but want to use that specific Linux distribution.

> Azure Batch pool allocation with AlmaLinux requires a workaround since native container tasks are not supported with this image.

## Architecture

The deployment creates the following resources:

* **Resource Group** - Container for all resources
* **Virtual Network & Subnet** - Network isolation for Batch nodes
* **Storage Account** - Hosts the start task script
* **Batch Account** - Manages the Batch pool and jobs
* **Batch Pool** - AlmaLinux 9 nodes with Docker CE installed via start task
* **Test Job** - Runs a `hello-world` Docker container to verify the setup

## Prerequisites

* [Terraform](https://www.terraform.io/downloads) >= 1.5
* [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli) authenticated (`az login`)
* An Azure subscription

## Providers

Make sure you have the following providers configured in your Terraform configuration file:

```hcl
terraform {
  required_version = ">= 1.5"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
    null = {
      source  = "hashicorp/null"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}
```

## Variables

Define the following variables in your Terraform configuration file:

```hcl
variable "location" {
  description = "Azure region for all resources"
  type        = string
  default     = "swedencentral"
}

variable "prefix" {
  description = "Prefix for resource names"
  type        = string
  default     = "batch-alma"
}

variable "batch_pool_vm_size" {
  description = "VM size for the Batch pool nodes"
  type        = string
  default     = "Standard_D2s_v3"
}

variable "batch_pool_node_count" {
  description = "Number of dedicated nodes in the Batch pool"
  type        = number
  default     = 1
}
```

## Resource Group and Networking

Create a resource group, virtual network, and subnet using the following Terraform configuration:

```hcl
resource "azurerm_resource_group" "this" {
  name     = "rg-${var.prefix}"
  location = var.location
}

resource "azurerm_virtual_network" "this" {
  name                = "vnet-${var.prefix}"
  location            = azurerm_resource_group.this.location
  resource_group_name = azurerm_resource_group.this.name
  address_space       = ["10.0.0.0/16"]
}

resource "azurerm_subnet" "batch" {
  name                 = "snet-batch"
  resource_group_name  = azurerm_resource_group.this.name
  virtual_network_name = azurerm_virtual_network.this.name
  address_prefixes     = ["10.0.1.0/24"]
}
```

## Storage Account

Create a storage account to host the start task script:

```hcl
resource "azurerm_storage_account" "this" {
  name                            = replace("st${var.prefix}", "-", "")
  location                        = azurerm_resource_group.this.location
  resource_group_name             = azurerm_resource_group.this.name
  account_tier                    = "Standard"
  account_replication_type        = "LRS"
  allow_nested_items_to_be_public = false
}

resource "azurerm_storage_container" "scripts" {
  name               = "scripts"
  storage_account_id = azurerm_storage_account.this.id
}

resource "azurerm_storage_blob" "start_task" {
  name                   = "start-task.sh"
  storage_account_name   = azurerm_storage_account.this.name
  storage_container_name = azurerm_storage_container.scripts.name
  type                   = "Block"
  source                 = "${path.module}/start-task.sh"
}

data "azurerm_storage_account_sas" "this" {
  connection_string = azurerm_storage_account.this.primary_connection_string
  https_only        = true
  start             = timestamp()
  expiry            = timeadd(timestamp(), "8760h") # 1 year

  resource_types {
    service   = false
    container = false
    object    = true
  }

  services {
    blob  = true
    queue = false
    table = false
    file  = false
  }

  permissions {
    read    = true
    write   = false
    delete  = false
    list    = false
    add     = false
    create  = false
    update  = false
    process = false
    tag     = false
    filter  = false
  }
}
```

## Start Task Script

Create a `start-task.sh` file that installs Docker CE on AlmaLinux 9:

```bash
#!/bin/bash
set -euo pipefail

# Azure Batch start task for AlmaLinux 9: install Docker CE
# This fixes "docker not found" issues on custom AlmaLinux images

echo ">>> Installing Docker CE on AlmaLinux 9..."

# Install required dependencies
dnf install -y dnf-utils device-mapper-persistent-data lvm2

# Add Docker's official repository
dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo

# Install Docker CE
dnf install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Enable and start Docker
systemctl enable docker
systemctl start docker

# Verify the installation
docker info

echo ">>> Docker CE installed and running."
```

## Batch Account

Create the Azure Batch account:

```hcl
resource "azurerm_batch_account" "this" {
  name                                = replace("ba${var.prefix}", "-", "")
  location                            = azurerm_resource_group.this.location
  resource_group_name                 = azurerm_resource_group.this.name
  pool_allocation_mode                = "BatchService"
  storage_account_id                  = azurerm_storage_account.this.id
  storage_account_authentication_mode = "StorageKeys"
}
```

## Batch Pool with AlmaLinux 9 and Docker

Now create the Batch pool with AlmaLinux 9 and Docker CE:

```hcl
resource "azurerm_batch_pool" "this" {
  name                = "alma-pool"
  resource_group_name = azurerm_resource_group.this.name
  account_name        = azurerm_batch_account.this.name
  vm_size             = var.batch_pool_vm_size
  node_agent_sku_id   = "batch.node.el 9"
  display_name        = "AlmaLinux Docker Pool"

  fixed_scale {
    target_dedicated_nodes = var.batch_pool_node_count
  }

  storage_image_reference {
    publisher = "almalinux"
    offer     = "almalinux-x86_64"
    sku       = "9-gen2"
    version   = "latest"
  }

  container_configuration {
    type = "DockerCompatible"
  }

  network_configuration {
    subnet_id = azurerm_subnet.batch.id
  }

  start_task {
    command_line       = "bash -c 'chmod +x start-task.sh && ./start-task.sh'"
    wait_for_success   = true
    task_retry_maximum = 3
    user_identity {
      auto_user {
        elevation_level = "Admin"
        scope           = "Pool"
      }
    }

    resource_file {
      http_url  = "${azurerm_storage_blob.start_task.url}${data.azurerm_storage_account_sas.this.sas}"
      file_path = "start-task.sh"
    }
  }
}
```

> Notice we're using `batch.node.el 9` as the node agent SKU and `almalinux:almalinux-x86_64:9-gen2` as the image. The start task runs on each node to install Docker CE from the official Docker repository.

## Why Native Container Tasks Don't Work (ContainerPoolNotSupported)

Azure Batch has two ways to run containers:

1. **Native container tasks** - Using `containerSettings` in the task JSON
2. **Command-line Docker** - Running `docker run` directly in the task command

AlmaLinux does not support native container tasks. If you try to use `containerSettings`, you'll get:

```
ContainerPoolNotSupported: The specified pool does not support container tasks.
```

This happens because Azure Batch's native container support only works with specific VM images that have Batch-integrated container runtimes (typically `microsoft-azure-batch` publisher images or certain Ubuntu images).

### The Workaround

This sample uses command-line Docker instead. Tasks invoke `docker run` directly:

```json
{
  "id": "test-docker-task",
  "commandLine": "/bin/bash -c \"docker run --rm hello-world\"",
  "userIdentity": {
    "autoUser": {
      "elevationLevel": "admin",
      "scope": "pool"
    }
  }
}
```

This approach provides full Docker functionality on AlmaLinux.

## Test Job

Create a test job and task to verify Docker is working:

```hcl
resource "azurerm_batch_job" "test" {
  name          = "test-docker-job"
  batch_pool_id = azurerm_batch_pool.this.id
}

resource "null_resource" "test_task" {
  depends_on = [azurerm_batch_job.test]

  provisioner "local-exec" {
    command = <<-EOT
      az batch task create \
        --job-id "test-docker-job" \
        --json-file ${path.module}/test-task.json \
        --account-name ${azurerm_batch_account.this.name} \
        --account-endpoint ${azurerm_batch_account.this.account_endpoint}
    EOT
  }
}
```

Create a `test-task.json` file:

```json
{
  "id": "test-docker-task",
  "commandLine": "/bin/bash -c \"docker run --rm hello-world\"",
  "userIdentity": {
    "autoUser": {
      "elevationLevel": "admin",
      "scope": "pool"
    }
  }
}
```

## Deploy the Infrastructure

Deploy the infrastructure using the following commands:

```bash
cd deploy
terraform init
terraform apply
```

## Verify the Deployment

After deployment, verify the test task output:

```bash
az batch task show \
  --job-id test-docker-job \
  --task-id test-docker-task \
  --account-name <batch_account_name> \
  --account-endpoint <batch_account_name>.<location>.batch.azure.com
```

## Clean Up

To destroy all resources:

```bash
terraform destroy
```

Hope it helps!

**References:**

* [Sample Code](https://github.com/cmendible/azure.samples/tree/main/azure_batch_alma_linux)
* [Use Azure Batch to run container workloads](https://learn.microsoft.com/azure/batch/batch-docker-container-workloads)
* [Deploy an Azure Batch account and two pools with a start task - Terraform](https://learn.microsoft.com/azure/batch/quick-deploy-batch-account-two-pools-start-task-terraform)
* [Jobs and tasks in Azure Batch - Start Task](https://learn.microsoft.com/azure/batch/jobs-and-tasks#tasks)
