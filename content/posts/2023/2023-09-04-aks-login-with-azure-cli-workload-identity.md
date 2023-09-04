---
author: Carlos Mendible
categories:
- azure
- kubernetes
date: "2023-09-04T10:00:00Z"
description: 'AKS: Login with Azure CLI and Workload Identity'
images: ["/assets/img/posts/aks.png"]
draft: false
tags: ["aks", "terraform", "az cli", "workload identity", "aad", "azure active directory"]
title: 'AKS: Login with Azure CLI and Workload Identity'
---

In this post I'll show you how to setup Workload Identity in an AKS cluster using terraform and then deploy a pod with Azure CLI that you will use to login to Azure.

**Long story short**: once workload identity is configured and enabled, kubernetes will inject 3 environment variables needed to login with Azure CLI:

* AZURE_FEDERATED_TOKEN_FILE
* AZURE_CLIENT_ID
* AZURE_TENANT_ID

And then you can login with Azure CLI using the following command:

``` shell
az login --federated-token "$(cat  $AZURE_FEDERATED_TOKEN_FILE)" --service-principal -u $AZURE_CLIENT_ID -t $AZURE_TENANT_ID
```

Now let's go step by step:

## Using Terraform to create an AKS cluster with workload identity

### Create variables.tf with the following contents

``` terraform
# Resource Group Name
variable "resource_group_name" {
  default = "aks-workload-identity"
}

# Location of the services
variable "location" {
  default = "West Europe"
}

# Name of the AKS cluster
variable "cluster_name" {
  default = "aks-cfm"
}

# DNS prefix for the AKS cluster
variable "dns_prefix" {
  default = "aks-cfm"
}

# Log Analytics Workspace Name
variable "log_workspace_name" {
  default = "aks-cfm-logs"
}

# Managed Identity Name
variable "managed_identity_name" {
  default = "aks-workload-identity"
}
```

### Create providers.tf with the following contents

``` terraform
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "3.44.1"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "2.18.0"
    }
  }
}

terraform {
  required_version = ">= 1.1.8"
}

provider "azurerm" {
  features {}
}

# Configuring the kubernetes provider
provider "kubernetes" {
  host                   = azurerm_kubernetes_cluster.k8s.kube_config.0.host
  client_certificate     = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.client_certificate)
  client_key             = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.client_key)
  cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.cluster_ca_certificate)
}
```

### Create main.tf with the following contents

``` terraform
data "azurerm_subscription" "current" {}

data "azurerm_client_config" "current" {}

# Create Resource Group
resource "azurerm_resource_group" "rg" {
  name     = var.resource_group_name
  location = var.location
}
```

### Create mi.tf with the following contents

``` terraform
# Create Managed Identity
resource "azurerm_user_assigned_identity" "mi" {
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location

  name = var.managed_identity_name
}

# Assign the Reader role to the Managed Identity
resource "azurerm_role_assignment" "reader" {
  scope                = azurerm_resource_group.rg.id
  role_definition_name = "Reader"
  principal_id         = azurerm_user_assigned_identity.mi.principal_id
}

# Associate the Managed Identity with the AKS cluster
resource "azurerm_federated_identity_credential" "federation" {
  name                = "aks-workload-identity"
  resource_group_name = azurerm_resource_group.rg.name
  audience            = ["api://AzureADTokenExchange"]
  issuer              = azurerm_kubernetes_cluster.k8s.oidc_issuer_url
  parent_id           = azurerm_user_assigned_identity.mi.id
  subject             = "system:serviceaccount:default:workload-identity-test-account"
}
```

> Note: A federated identity credential is created to associate the managed identity with the `workload-identity-test-account` Service Account inside the AKS cluster.

### Create log_analytics.tf with the following contents

``` terraform
resource "azurerm_log_analytics_workspace" "logs" {
  name                = var.log_workspace_name
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
}
```

### Create vnet.tf with the following contents

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

### Create aks.tf with the following contents

``` terraform
# Deploy Kubernetes
resource "azurerm_kubernetes_cluster" "k8s" {
  name                      = var.cluster_name
  location                  = azurerm_resource_group.rg.location
  resource_group_name       = azurerm_resource_group.rg.name
  dns_prefix                = var.dns_prefix
  oidc_issuer_enabled       = true
  workload_identity_enabled = true
  role_based_access_control_enabled = true
  oms_agent {
    log_analytics_workspace_id = azurerm_log_analytics_workspace.logs.id
  }

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

# Create Service Account with the azure.workload.identity/client-id annotation
resource "kubernetes_service_account" "default" {
  metadata {
    name      = "workload-identity-test-account"
    namespace = "default"
    annotations = {
      "azure.workload.identity/client-id" = azurerm_user_assigned_identity.mi.client_id
    }
    labels = {
      "azure.workload.identity/use" : "true"
    }
  }
}
```

> Note: The cluster has the `oidc_issuer_enabled` set to true and the service account is associated with the managed identity created in the previous step via the `azure.workload.identity/client-id` annotation.

### Deploy the cluster

Run the following command:

``` shell
terraform init
terraform apply -auto-approve
```

## Deploy a pod with Azure CLI

### Create a pod.yaml with the following contents

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: az-cli
  namespace: default
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: workload-identity-test-account
  containers:
    - name: az-cli
      image: mcr.microsoft.com/azure-cli
      ports:
        - containerPort: 80
      command:
          - sh
          - -c
          - sleep 1d
```

> Note: The pod is using the service account created in the previous step and the label `azure.workload.identity/use` is set to `true` to enable workload identity.

### Deploy the Azure CLI deployment

Run the following commands:

``` shell
az aks get-credentials --resource-group aks-workload-identity --name aks-cfm
kubectl apply -f pod.yaml
```

## Test Azure CLI with Workload Identity

### Exec into the pod

Run the following command:

``` shell
kubectl exec -it az-cli -- /bin/bash
```

### Login with Azure CLI

Run the following command:

``` shell
az login --federated-token "$(cat  $AZURE_FEDERATED_TOKEN_FILE)" --service-principal -u $AZURE_CLIENT_ID -t $AZURE_TENANT_ID
```

> Note: Once workload identity is enabled, kubernetes will inject the 3 environment variables needed to login with Azure CLI.

If everything went ok, you should be able to work with Azure CLI without any issues.

Please find the complete samples [here](https://github.com/cmendible/azure.samples/tree/main/aks_workload_identity)

Hope it helps!

References:

* [Use Azure AD workload identity with Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview?tabs=dotnet)
