---
author: Carlos Mendible
categories:
- azure
- kubernetes
date: "2022-01-25T10:00:00Z"
description: 'AKS: Disable local accounts with Terraform'
images: ["/assets/img/posts/aks.png"]
draft: false
tags: ["aks", "terraform", "aad", "azure active directory"]
title: 'AKS: Disable local accounts with Terraform'
---

When deploying an AKS cluster, even if you configure RBAC or AAD integration, local accounts will be enabled by default. This means that, given the right set of permitions, a user will be able to run the `az get-credentials` command with the `--admin` flag which will give you a non-audtibale access to the cluster.

But don't worry! it's possible to disable local account while deploying a new cluster or update an existing one, through the `disable-local-accounts` flag when using Azure CLI.

> If you disable local accounts on an existing AKS cluster, and want to revoke access to any user, be sure to [rotate the cluster certificates](https://docs.microsoft.com/en-us/azure/aks/certificate-rotation#rotate-your-cluster-certificates) 

Once local accounts are disabled you'll find another challenge: when using service principals or managed identities to connect to the cluster, you'll need a non-interactive flow to get valid cluster credentials, otherwise when attempting to run commands against the cluster you'll get prompted for authentication.

[kubelogin](https://github.com/Azure/kubelogin) is a tool that will help you with the non-interactive flow by providing the `get-token` command, which you'll learn to use together with the exec credential plugin in the following sample.

## Using Terraform to create an AKS cluster with local accounts disabled.

### Create variables.tf with the following contents:

``` terraform	
# Location of the services
variable "location" {
  default = "west europe"
}

# Resource Group Name
variable "resource_group" {
  default = "aks-no-local-accounts"
}

# Name of the AKS cluster
variable "aks_name" {
  default = "aksnolocalaccounts"
}

# Name of the Service Princiapl we'll use to connect to the cluster.
variable "sp_name" {
  default = "aksnolocalaccounts"
}
```

### Create providers.tf with the following contents:

``` terraform
terraform {
  required_version = ">= 0.13.5"
  required_providers {
    azurerm = {
      version = "= 2.97.0"
    }
    azuread = {
      version = "= 1.4.0"
    }
    kubernetes = {
      version = "= 2.8.0"
    }
  }
}

provider "azurerm" {
  features {}
}

# Configuring the kubernetes provider
provider "kubernetes" {
  host                   = azurerm_kubernetes_cluster.aks.kube_config.0.host
  cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.aks.kube_config.0.cluster_ca_certificate)

  # Using kubelogin to get an AAD token for the cluster.
  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command = "kubelogin"
    args = [
      "get-token",
      "--environment",
      "AzurePublicCloud",
      "--server-id",
      data.azuread_service_principal.aks_aad_server.application_id, # Application Id of the Azure Kubernetes Service AAD Server.
      "--client-id",
      azuread_application.sp.application_id, // Application Id of the Service Principal we'll create via terraform.
      "--client-secret",
      random_password.passwd.result, // The Service Principal's secret.
      "-t",
      data.azurerm_subscription.current.tenant_id, // The AAD Tenant Id.
      "-l",
      "spn" // Login using a Service Principal..
    ]
  }
}
```

> In the `kubernetes` provider definition the `exec` plugin is used together with the `kubelogin get-token` command. 

### Create main.tf with the following contents:

``` terraform	
# Create Resource Group
resource "azurerm_resource_group" "rg" {
  name     = var.resource_group
  location = var.location
}

# Get current Subscription
data "azurerm_subscription" "current" {}

# Get current Client
data "azuread_client_config" "current" {}

# We'll need the Application Id of the Azure Kubernetes Service AAD Server.
data "azuread_service_principal" "aks_aad_server" {
  display_name = "Azure Kubernetes Service AAD Server"
}

# Create VNET for AKS
resource "azurerm_virtual_network" "vnet" {
  name                = "private-network"
  address_space       = ["10.0.0.0/8"]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

# Create the Subnet for AKS.
resource "azurerm_subnet" "aks" {
  name                 = "aks"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.240.0.0/16"]
}

# Create the AKS cluster.
# Cause this is a test node_count is set to 1 
resource "azurerm_kubernetes_cluster" "aks" {
  name                   = var.aks_name
  location               = azurerm_resource_group.rg.location
  resource_group_name    = azurerm_resource_group.rg.name
  dns_prefix             = var.aks_name
  local_account_disabled = true

  default_node_pool {
    name            = "default"
    node_count      = 1
    vm_size         = "Standard_D2s_v3"
    os_disk_size_gb = 30
    os_disk_type    = "Ephemeral"
    vnet_subnet_id  = azurerm_subnet.aks.id
  }

  # Using Managed Identity
  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin = "kubenet"
    network_policy = "calico"
  }

  role_based_access_control {
    enabled = true
    azure_active_directory {
      admin_group_object_ids = [
        azuread_group.aks_admins.object_id,
      ]
      azure_rbac_enabled = false
      managed            = true
      tenant_id          = data.azurerm_subscription.current.tenant_id
    }
  }
}

resource "kubernetes_namespace" "ns" {
  metadata {
    name = "nolocalaccounts"
  }
}
```

> Local accounts are disabled by using the `local_account_disabled` property. Also RBAC is enabled and an AAD group is configured as cluster administrator.

### Create service_principal.tf with the following contents:

``` terraform
resource "azuread_application" "sp" {
  display_name    = var.sp_name
}

resource "azuread_service_principal" "sp" {
  application_id = azuread_application.sp.application_id
}

# Generate password for the Service Principal
resource "random_password" "passwd" {
  length      = 32
  min_upper   = 4
  min_lower   = 2
  min_numeric = 4
  keepers = {
    aks_app_id = azuread_application.sp.id
  }
}

# Create kubecost's Service principal password
resource "azuread_service_principal_password" "sp_password" {
  service_principal_id = azuread_service_principal.sp.id
  value                = random_password.passwd.result
  end_date             = "2099-01-01T00:00:00Z"
}

resource "azuread_group" "aks_admins" {
  display_name     = "aks_admins"
  owners           = [data.azuread_client_config.current.object_id]

  members = [
    data.azuread_client_config.current.object_id,
    azuread_service_principal.sp.object_id,
  ]
}
```

> Once the Service Principal is created it is added to a AAD group (`aks_admins`) which is configured as cluster administrator in **main.tf**.

### Deploy the cluster:

As prerequisite first download [kubelogin](https://github.com/Azure/kubelogin):

``` powershell
Invoke-WebRequest https://github.com/Azure/kubelogin/releases/download/v0.0.11/kubelogin-win-amd64.zip -OutFile kubelogin.zip
Expand-Archive .\kubelogin.zip
mv .\kubelogin\bin\windows_amd64\kubelogin.exe .\kubelogin.exe
rm .\kubelogin\
rm .\kubelogin.zip
```

Now run the following commands:

``` shell
terraform init
terraform apply -auto-approve
```

If everything is ok, the deployment will finish without issues and you'll be able to check for the existence of the `nolocalaccounts` namespace in your cluster.

Please find the complete samples [here](https://github.com/cmendible/azure.samples/tree/main/aks_disable_local_accounts)

References:

* [Disable local accounts](https://docs.microsoft.com/en-us/azure/aks/managed-aad#disable-local-accounts)
* [kubelogin](https://github.com/Azure/kubelogin)
