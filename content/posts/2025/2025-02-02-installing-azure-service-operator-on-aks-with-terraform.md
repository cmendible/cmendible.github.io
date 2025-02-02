---
author: Carlos Mendible
categories:
- azure
- kubernetes
date: "2025-02-02"
description: 'Installing Azure Service Operator on AKS with Terraform'
draft: false
tags: ["aso", "terraform"]
title: 'Installing Azure Service Operator on AKS with Terraform'
---

In this post, we will go through the steps required to install the Azure Service Operator (ASO) on an Azure Kubernetes Service (AKS) cluster using Terraform. The Azure Service Operator is an open-source project that enables you to provision and manage Azure resources using Kubernetes custom resources.

To install the Azure Service Operator on an AKS cluster, follow these steps:

## Providers

Make sure you have the following providers configured in your Terraform configuration file:

```bash
terraform {
  required_version = "> 0.12"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "4.17.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "2.18.0"
    }
    azuread = {
      source  = "hashicorp/azuread"
      version = "3.1.0"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "2.17.0"
    }
  }
}

provider "azurerm" {
  features {}
}

provider "azuread" {
}

# Used to create some namespaces
provider "kubernetes" {
  host                   = azurerm_kubernetes_cluster.k8s.kube_config.0.host
  client_certificate     = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.client_certificate)
  client_key             = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.client_key)
  cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.cluster_ca_certificate)
}

# Used to install Cert Manager and the Azure Service Operator
provider "helm" {
  kubernetes {
    host                   = azurerm_kubernetes_cluster.k8s.kube_config.0.host
    client_certificate     = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.client_certificate)
    client_key             = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.client_key)
    cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.cluster_ca_certificate)
  }
}
```

## Variables

Define the following variables in your Terraform configuration file:

```bash
variable "resource_group_name" {
  default = "rg-aso-demo"
}

variable "location" {
  default = "northeurope"
}

variable "cluster_name" {
  default = "aks-aso"
}

variable "dns_prefix" {
  default = "aks-aso"}

variable "managed_identity_name" {
  default = "aks-workload-identity"
}
```

## Resource Group

Create a resource group using the following Terraform configuration:

```bash
# Get current subscription
data "azurerm_subscription" "current" {}

# Get current client
data "azurerm_client_config" "current" {}

#  Create Resource Group
resource "azurerm_resource_group" "rg" {
  name     = var.resource_group_name
  location = var.location
}
```

## Virtual Network

Create a virtual network and subnet using the following Terraform configuration:

```bash
resource "azurerm_virtual_network" "vnet" {
  name                = "vnet-aks-aso"
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

## AKS Cluster

Create an AKS cluster using the following Terraform configuration:

```bash
# Deploy Kubernetes
resource "azurerm_kubernetes_cluster" "k8s" {
  name                = var.cluster_name
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = var.dns_prefix

  oidc_issuer_enabled               = true # Required for workload identity
  workload_identity_enabled         = true # Required for workload identity
  role_based_access_control_enabled = true

  default_node_pool {
    name                 = "default"
    node_count           = 2
    vm_size              = "Standard_D2s_v3"
    os_disk_size_gb      = 30
    os_disk_type         = "Ephemeral"
    vnet_subnet_id       = azurerm_subnet.aks-subnet.id
    max_pods             = 15
    auto_scaling_enabled = false

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
    network_policy      = "cilium"
    network_data_plane  = "cilium"
  }
}

# Create Managed Identity
resource "azurerm_user_assigned_identity" "mi" {
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location

  name = var.managed_identity_name
}

# Assign the Managed Identity the Contributor role on the target subscription
resource "azurerm_role_assignment" "mi" {
  scope                = data.azurerm_subscription.current.id
  role_definition_name = "Contributor"
  principal_id         = azurerm_user_assigned_identity.mi.principal_id
}

# Create the Federated Identity Credential. ASO uses the `azureserviceoperator-default` service account
resource "azurerm_federated_identity_credential" "federation" {
  name                = "aks-workload-identity"
  resource_group_name = azurerm_resource_group.rg.name
  audience            = ["api://AzureADTokenExchange"]
  issuer              = azurerm_kubernetes_cluster.k8s.oidc_issuer_url
  parent_id           = azurerm_user_assigned_identity.mi.id
  subject             = "system:serviceaccount:azureserviceoperator-system:azureserviceoperator-default"
}
```

## Azure Service Operator

Install the Azure Service Operator using the following Terraform configuration:

```bash
# Create the azureserviceoperator-system namespace in k8s
resource "kubernetes_namespace" "azure-service-operator" {
  metadata {
    name = "azureserviceoperator-system"
  }
}

# Create the cert-manager namespace in k8s
resource "kubernetes_namespace" "cert-manager" {
  metadata {
    name = "cert-manager"
  }
}

# Install the cert-manager helm chart
resource "helm_release" "cert-manager" {
  name       = "cert-manager"
  chart      = "cert-manager"
  version    = "1.16.3"
  namespace  = kubernetes_namespace.cert-manager.metadata.0.name
  repository = "https://charts.jetstack.io"

  set {
    name  = "crds.enabled" # Install CRDs
    value = "true"
  }
}

# Install the azure-service-operator helm chart
resource "helm_release" "azure-service-operator" {
  name       = "aso"
  chart      = "azure-service-operator"
  version    = "2.11.0"
  namespace  = kubernetes_namespace.azure-service-operator.metadata.0.name
  repository = "https://raw.githubusercontent.com/Azure/azure-service-operator/refs/heads/main/v2/charts/"

  set {
    name  = "crdPattern" # Installing all CRDs
    value = "*"
  }

  set {
    name  = "azureSubscriptionID" # Set the target Azure Subscription ID
    value = data.azurerm_subscription.current.subscription_id
  }

  set {
    name  = "azureTenantID" # Set the Azure Tenant ID
    value = data.azurerm_subscription.current.tenant_id
  }

  set {
    name  = "azureClientID" # Set the Managed Identity Client ID
    value = azurerm_user_assigned_identity.mi.client_id
  }

  set {
    name  = "useWorkloadIdentityAuth" # Use the workload identity
    value = "true"
  }

  depends_on = [
    helm_release.cert-manager # Wait for cert-manager to be installed
  ]
}
```

## Deploy a Resource Group using Azure Service Operator (ASO)

Create a Kubernetes custom resource to deploy a resource group using the Azure Service Operator:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: resources.azure.com/v1api20200601
kind: ResourceGroup
metadata:
  name: aso-test-resource-group
  namespace: default
spec:
  location: westeurope
EOF
```

Check the status of the resource group deployment using the following command:

```bash
kubectl get resourcegroup aso-test-resource-group -n default
```

If reason is `Succeeded`, the resource group has been successfully deployed. You can also check the status of the resource group deployment using the Azure portal or the Azure CLI.

Hope it helps!

**References:**

* [Azure Service Operator](https://azure.github.io/azure-service-operator/)
