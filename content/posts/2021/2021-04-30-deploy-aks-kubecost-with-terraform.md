---
author: Carlos Mendible
categories:
- azure
- kubernetes
crosspost_to_medium: true
date: "2021-04-30T19:00:00Z"
description: 'Deploy AKS + Kubecost with Terraform'
images: ["/assets/img/posts/aks_kubecost.png"]
draft: false
tags: ["terraform", "kubecost"]
title: 'Deploy AKS + Kubecost with Terraform'
---

This morning I saw this tweet from Mr Brendan Burns:

{{< tweet 1387933511433154564 >}}

And I'm sure that once you also read through it, you'll learn that you have to take several steps in order to achieve [AKS Cost Monitoring and Governance With Kubecost](http://blog.kubecost.com/blog/aks-cost/).

I'm going to try and save you some time, providing you with a basic terraform configuration to help you get up and running in a breeze.

> If you want to learn more about [Kubecost](https://www.kubecost.com/) in the context of AKS and Azure please read the [Cost Governance](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/scenarios/aks/eslz-security-governance-and-compliance#cost-governance) section of the [AKS enterprise-scale platform security governance and compliance](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/scenarios/aks/eslz-security-governance-and-compliance#cost-governance) guidelines.

## Deploy AKS and [Kubecost](https://www.kubecost.com/) with Terraform

### 1. Create a `provider.tf` file with the following contents:

``` yaml
terraform {
  required_version = "> 0.14"
  required_providers {
    azurerm = {
      version = "= 2.57.0"
    }
    azuread = {
      version = "= 1.4.0"
    }
    kubernetes = {
      version = "= 2.1.0"
    }
    helm = {
      version = "= 2.1.2"
    }
  }
}

provider "azurerm" {
  features {}
}

# Configuring the kubernetes provider
# AKS resource name is aks: azurerm_kubernetes_cluster.aks
provider "kubernetes" {
  host                   = azurerm_kubernetes_cluster.aks.kube_config.0.host
  client_certificate     = base64decode(azurerm_kubernetes_cluster.aks.kube_config.0.client_certificate)
  client_key             = base64decode(azurerm_kubernetes_cluster.aks.kube_config.0.client_key)
  cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.aks.kube_config.0.cluster_ca_certificate)
}

# Configuring the helm provider
# AKS resource name is aks: azurerm_kubernetes_cluster.aks
provider "helm" {
  kubernetes {
    host                   = azurerm_kubernetes_cluster.aks.kube_config.0.host
    client_certificate     = base64decode(azurerm_kubernetes_cluster.aks.kube_config.0.client_certificate)
    client_key             = base64decode(azurerm_kubernetes_cluster.aks.kube_config.0.client_key)
    cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.aks.kube_config.0.cluster_ca_certificate)
  }
}
```

Note that you'll be using `azurerm` to deploy Azure services, `azuread` to create a Service Principal information and the `kubernetes` and `helm` provider to install [Kubecost](https://www.kubecost.com/).

### 2. Create a `variables.tf` file with the following contents:

``` yaml
# Location of the services
variable "location" {
  default = "west europe"
}

# Resource Group Name
variable "resource_group" {
  default = "aks-kubecost"
}

# Name of the AKS cluster
variable "aks_name" {
  default = "aksmsftkubecost"
}

# Name of the Servic Principal used by Kubecost
variable "kubecost_sp_name" {
  default = "kubecost"
}
```

Note: Replace the the default values with your desired location and names.

### 3. Create a `main.tf` file with the following contents:

``` yaml
# Create Resource Group
resource "azurerm_resource_group" "rg" {
  name     = var.resource_group
  location = var.location
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
  name                = var.aks_name
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = var.aks_name

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
  }

  addon_profile {
    kube_dashboard {
      enabled = false
    }
  }
}

# Create Application registration for Kubecost
resource "azuread_application" "kubecost" {
  display_name               = var.kubecost_sp_name
  identifier_uris            = ["http://${var.kubecost_sp_name}"]
}

# Create Service principal for kubecost
resource "azuread_service_principal" "kubecost" {
  application_id = azuread_application.kubecost.application_id
}

# Generate password for the Service Principal
resource "random_password" "passwd" {
  length      = 32
  min_upper   = 4
  min_lower   = 2
  min_numeric = 4
   keepers = {
    aks_app_id = azuread_application.kubecost.id
  }
}

# Create kubecost's Service principal password
resource "azuread_service_principal_password" "main" {
  service_principal_id = azuread_service_principal.kubecost.id
  value                = random_password.passwd.result
  end_date             = "2099-01-01T00:00:00Z"
}

# Get current Subscription
data "azurerm_subscription" "current" {
}

# Create kubecost custom role
resource "azurerm_role_definition" "kubecost" {
  name        = "kubecost_rate_card_query"
  scope       = data.azurerm_subscription.current.id
  description = "kubecost Rate Card query role"

  permissions {
    actions     = [
      "Microsoft.Compute/virtualMachines/vmSizes/read",
      "Microsoft.Resources/subscriptions/locations/read",
      "Microsoft.Resources/providers/read",
      "Microsoft.ContainerService/containerServices/read",
      "Microsoft.Commerce/RateCard/read",
    ]
    not_actions = []
  }

  assignable_scopes = [
    data.azurerm_subscription.current.id
  ]
}

# Assign kubecost's custom role at the subscription level
resource "azurerm_role_assignment" "kubecost" {
  scope                = data.azurerm_subscription.current.id
  role_definition_name = azurerm_role_definition.kubecost.name
  principal_id         = azuread_service_principal.kubecost.object_id
}
```

### 4. Create a `kubecost.tf` file with the following contents:

``` yaml
# Create the kubecost namespace
resource "kubernetes_namespace" "kubecost" {
  metadata {
    name = "kubecost"
  }
}

# Install kubecost using the hem chart
resource "helm_release" "kubecost" {
  name       = "kubecost"
  chart      = "cost-analyzer"
  namespace  = "kubecost"
  version    = "1.79.1"
  repository = "https://kubecost.github.io/cost-analyzer/"

  # Set the cluster name
  set {
    name  = "kubecostProductConfigs.clusterName"
    value = var.aks_name
  }

  # Set the currency
  set {
    name  = "kubecostProductConfigs.currencyCode"
    value = "EUR"
  }

  # Set the region
  set {
    name  = "kubecostProductConfigs.azureBillingRegion"
    value = "NL"
  }
  
  # Generate a secret based on the Azure configuration provided below
  set {
    name  = "kubecostProductConfigs.createServiceKeySecret"
    value = true
  }

  # Azure Subscription ID
  set {
    name  = "kubecostProductConfigs.azureSubscriptionID"
    value = data.azurerm_subscription.current.id
  }

  # Azure Client ID
  set {
    name  = "kubecostProductConfigs.azureClientID"
    value = azuread_application.kubecost.application_id
  }

  # Azure Client Password
  set {
    name  = "kubecostProductConfigs.azureClientPassword"
    value = random_password.passwd.result
  }

  # Azure Tenant ID
  set {
    name  = "kubecostProductConfigs.azureTenantID"
    value = data.azurerm_subscription.current.tenant_id
  }
}
```

The configuration in the previous file installs [Kubecost](https://www.kubecost.com/) in the AKS cluster. If you want to learn more about the available configuration options please check the following file: [values.yaml](https://github.com/kubecost/cost-analyzer-helm-chart/blob/master/cost-analyzer/values.yaml) 

### 5. Deploy the solution:

Run the following commands:

``` shell
terraform init
terraform plan -out tf.plan
terraform apply ./tf.plan
```

### 6. Test and browse Kubecost:

To check the status of the kubecost pods run: 

``` shell 
az aks get-credentials -g aks-kubecost -n aksmsftkubecost
kubectl get pods -n kubecost
```

Then run:

``` shell 
kubectl port-forward -n kubecost svc/kubecost-cost-analyzer 9090:9090
```

and browse to [http://localhost:9090](http://localhost:9090) so you can start learning!

Hope it helps! Please find the complete code [here](https://github.com/cmendible/azure.samples/tree/main/aks_kubecost)