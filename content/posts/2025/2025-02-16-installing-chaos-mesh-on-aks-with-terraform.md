---
author: Carlos Mendible
categories:
- azure
- kubernetes
date: "2025-02-16"
description: 'Installing Chaos Mesh on AKS with Terraform'
images: ["/assets/img/posts/kubernetes.png"]
draft: false
tags: ["chaos-mesh", "terraform"]
title: 'Installing Chaos Mesh on AKS with Terraform'
---

In this post, we will go through the steps required to install [Chaos Mesh](https://chaos-mesh.org/) on an Azure Kubernetes Service (AKS) cluster using Terraform.

[Chaos Mesh](https://chaos-mesh.org/) is a cloud-native Chaos Engineering platform that orchestrates chaos on Kubernetes environments. It is designed to be a scalable and extensible platform for chaos engineering.

> [Chaos Mesh](https://chaos-mesh.org/) is required if you want to use [Azure Chaos Studio](https://learn.microsoft.com/en-us/azure/chaos-studio/chaos-studio-overview) to run chaos experiments on your AKS clusters.

To install `Chaos Mesh` on an AKS cluster, follow these steps:

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

provider "kubernetes" {
  host                   = azurerm_kubernetes_cluster.k8s.kube_config.0.host
  client_certificate     = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.client_certificate)
  client_key             = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.client_key)
  cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.cluster_ca_certificate)
}

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
  default = "rg-chaos-mesh-demo"
}

variable "location" {
  default = "spaincentral"
}

variable "cluster_name" {
  default = "aks-chaos-mesh"
}

variable "dns_prefix" {
  default = "aks-chaos-mesh"
}
```

## Resource Group

Create a resource group using the following Terraform configuration:

```bash
resource "azurerm_resource_group" "rg" {
  name     = var.resource_group_name
  location = var.location
}
```

## Virtual Network

Create a virtual network and subnet using the following Terraform configuration:

```bash
resource "azurerm_virtual_network" "vnet" {
  name                = "vnet-aks-chaos-mesh"
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

  oidc_issuer_enabled               = true
  workload_identity_enabled         = true
  role_based_access_control_enabled = true

  # Enable Application routing add-on with NGINX features
  web_app_routing {
    dns_zone_ids = []
  }

  default_node_pool {
    name                 = "default"
    node_count           = 3
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

resource "azurerm_role_assignment" "kubelet_network_contributor" {
  scope                = azurerm_virtual_network.vnet.id
  role_definition_name = "Network Contributor"
  principal_id         = azurerm_kubernetes_cluster.k8s.identity[0].principal_id
}

resource "azurerm_role_assignment" "kubelet_network_reader" {
  scope                = azurerm_virtual_network.vnet.id
  role_definition_name = "Reader"
  principal_id         = azurerm_kubernetes_cluster.k8s.identity[0].principal_id
}
```

> The sample requires an ingress controller to be deployed in the cluster. The ingress controller is used to route traffic to the `Chaos Mesh` Dashboard. That's why we enable the `web_app_routing` add-on.

## Chaos Mesh

Install `Chaos Mesh` using the following Terraform configuration:

```bash
# Install the chaos mesh helm chart
resource "helm_release" "chaos" {
  create_namespace = true
  name             = "chaos-mesh"
  chart            = "chaos-mesh"
  namespace        = "chaos-testing"
  repository       = "https://charts.chaos-mesh.org"

  set {
    name  = "chaosDaemon.runtime"
    value = "containerd"
  }

  set {
    name  = "chaosDaemon.socketPath"
    value = "/run/containerd/containerd.sock"
  }

  # Enable FilterNamespace
  # Annotate namespaces to enable chaos experiments: kubectl annotate ns $NAMESPACE chaos-mesh.org/inject=enabled
  # https://chaos-mesh.org/docs/configure-enabled-namespace/#enable-filternamespace
  set {
    name  = "controllerManager.enableFilterNamespace"
    value = "true"
  }
}
```

## Apply the Terraform configuration

Apply the Terraform configuration using the following commands:

```bash
terraform init
export ARM_SUBSCRIPTION_ID="<your-subscription-id>"
terraform apply
```

## Get the cluster credentials

Get the cluster credentials using the following command:

```bash
az aks get-credentials --resource-group rg-chaos-mesh-demo --name aks-chaos-mesh --overwrite-existing
```

## Deploy the Ingress object for the Chaos Mesh Dashboard 

Create an `Ingress` using the following command:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: chaos-ingress
  namespace: chaos-testing
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: webapprouting.kubernetes.azure.com
  rules:
    - http:
        paths:
          - path: /chaos-mesh/?(.*)
            pathType: Prefix
            backend:
              service:
                name: chaos-dashboard
                port:
                  number: 2333
EOF
```

## Create the RBAC and service account to access the Chaos Mesh Dashboard

Now you can create a simple web application using the following command:

```bash
cat <<EOF | kubectl apply -f -
kind: ServiceAccount
apiVersion: v1
metadata:
  namespace: default
  name: account-cluster-manager

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: role-cluster-manager
rules:
- apiGroups: [""]
  resources: ["pods", "namespaces"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["chaos-mesh.org"]
  resources: [ "*" ]
  verbs: ["get", "list", "watch", "create", "delete", "patch", "update"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: bind-cluster-manager
subjects:
- kind: ServiceAccount
  name: account-cluster-manager
  namespace: default
roleRef:
  kind: ClusterRole
  name: role-cluster-manager
  apiGroup: rbac.authorization.k8s.io
EOF
```

Now it's time to access the `Chaos Mesh` Dashboard:

```bash
kubectl create token account-cluster-manager
# Copy the token
IP_ADDRESS=$(kubectl get service -n app-routing-system nginx -o jsonpath="{.status.loadBalancer.ingress[0].ip}")
# Browse to the Chaos Mesh Dashboard and paste the token to login
echo "http://$IP_ADDRESS/chaos-mesh"
```

> Remember to add the annotation `chaos-mesh.org/inject=enabled` to the namespaces where you want to run chaos experiments.

Hope it helps!

**References:**

* [Chaos Mesh](https://chaos-mesh.org/)
* [Chaos Studio](https://learn.microsoft.com/en-us/azure/chaos-studio/chaos-studio-overview)
