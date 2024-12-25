---
author: Carlos Mendible
categories:
- azure
- kubernetes
date: "2024-12-25"
description: 'AKS: Static Egress Gateway with Terraform'
images: ["/assets/img/posts/aks.png"]
draft: false
tags: ["aks", "containers", "terraform"]
title: 'AKS: Static Egress Gateway with Terraform'
---

Let's learn how to create an AKS cluster and enable Static Egress Gateway with Terraform. 

Static Egress Gateway in AKS provides a solution for configuring fixed source IP addresses for outbound traffic from your AKS workloads. This means you can use a specific range for egress traffic from specific workloads, whcih can be useful for scenarios like whitelisting IP addresses in a firewall.

> Note: Since at the time of writing Static Egress Gateway is a preview feature, we will use the [azapi](https://registry.terraform.io/providers/azure/azapi/latest/docs) provider to enable it.

## Prerequisites

Register the Static Egress Gateway preview feature:

``` bash
az feature register --namespace "Microsoft.ContainerService" --name "StaticEgressGatewayPreview"
``` 

Wait for the feature registration to complete:

``` bash
az feature show --namespace "Microsoft.ContainerService" --name "StaticEgressGatewayPreview"
```

## Creating an AKS cluster and enable Static Egress Gateway

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
    kubectl = {
      source = "gavinbunney/kubectl" # We'll use this provider to deploy the Static Egress Gateway manifest
    }
  }
}

provider "azapi" {
}

provider "azurerm" {
  features {}
}

provider "kubectl" {
  host                   = azurerm_kubernetes_cluster.k8s.kube_config.0.host
  client_certificate     = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.client_certificate)
  client_key             = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.client_key)
  cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.cluster_ca_certificate)
}

# The Resource Group Name
variable "resource_group_name" {
  description = "The name of the resource group"
  type        = string
  default     = "aks-static-egress-gateway"
}

# The AKS Cluster Name
variable "cluster_name" {
  description = "The name of the AKS cluster"
  type        = string
  default     = "aks-static-egress-gateway"
}

# Create the Resource Group
resource "azurerm_resource_group" "rg" {
  name     = var.resource_group_name
  location = "Spain Central"
}

# Create the AKS Cluster
resource "azurerm_kubernetes_cluster" "k8s" {
  name                = var.cluster_name
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = var.cluster_name
  default_node_pool {
    name                         = "default"
    node_count                   = 1
    vm_size                      = "Standard_DS2_v2"
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
    type = "SystemAssigned"
  }
}

# Enable Static Egress Gateway feature using the azapi provider
resource "azapi_update_resource" "enable_static_egress_gateway" {
  type                    = "Microsoft.ContainerService/managedClusters@2024-09-02-preview"
  resource_id             = azurerm_kubernetes_cluster.k8s.id
  ignore_missing_property = true
  body = {
    properties = {
      networkProfile = {
        staticEgressGatewayProfile = {
          enabled = true
        }
      }
    }
  }
}

# Create a Gateway Node Pool using the azapi provider
resource "azapi_resource" "gateway_node_pool" {
  type      = "Microsoft.ContainerService/managedClusters/agentPools@2024-09-02-preview"
  parent_id = azurerm_kubernetes_cluster.k8s.id
  name      = "gatewaypool"

  body = {
    properties = {
      mode              = "Gateway" # This is the new mode for Gateway Node Pools
      osType            = "Linux"
      vmSize            = "Standard_DS2_v2"
      count             = 2
      enableAutoScaling = false
      gatewayProfile = {
        publicIPPrefixSize = 31
      },
    }
  }
  depends_on = [azapi_update_resource.enable_static_egress_gateway] # Making sure the Static Egress Gateway is enabled before creating the Gateway Node Pool
}

# Create the Static Egress Gateway Configuration
resource "kubectl_manifest" "static_gateway_configuration" {
  yaml_body = <<YAML
    apiVersion: egressgateway.kubernetes.azure.com/v1alpha1
    kind: StaticGatewayConfiguration
    metadata:
      name: static-ip-gateway
      namespace: default
    spec:
      gatewayNodepoolName: ${azapi_resource.gateway_node_pool.name}
      provisionPublicIps: false
      excludeCidrs: 
      - ${azurerm_kubernetes_cluster.k8s.network_profile[0].pod_cidr}
      - ${azurerm_kubernetes_cluster.k8s.network_profile[0].service_cidr}
      - 169.254.169.254/32
    YAML
  depends_on = [
    azapi_update_resource.enable_static_egress_gateway,
    azapi_resource.gateway_node_pool
  ] # Making sure the Static Egress Gateway is enabled and the pool is created before creating the StaticGatewayConfiguration
}
```

> Note: We set the `provisionPublicIps` property to false to avoid provisioning public IPs for the gateway nodes. Also we exclude the AKS pod and service CIDRs and the Azure Instance Metadata Service CIDR from the Static Egress Gateway.

Run the following commands to create the AKS cluster and enable Static Egress Gateway:

``` bash
export ARM_SUBSCRIPTION_ID=<your-subscription-id>
terraform init
terraform apply
```

> Note: once the cluster is created, you can check the gateway pool nodes which are tainted with `kubernetes.azure.com/mode=gateway:NoSchedule` to make sure you don't deploy workloads on them.

``` bash

## Check the Static Egress Gateway configuration

Run the following commands to check the Static Egress Gateway configuration and verfy the provisioned private IPs:

``` bash
az aks get-credentials --resource-group aks-static-egress-gateway --name aks-static-egress-gateway
kubectl get StaticGatewayConfiguration static-ip-gateway
```

## Deploy a pod to the cluster that uses the Static Egress Gateway

Create a file called `pod.yaml` with the following contents:

``` bash
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: test-pod
  annotations:
    kubernetes.azure.com/static-gateway-configuration: static-ip-gateway # Use the annotation to specify the Static Egress Gateway configuration
  name: test-pod
  namespace: default
spec:
  containers:
  - image: busybox
    name: test-pod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Run the following commands to deploy the pod to the cluster:

``` bash
kubectl apply -f pod.yaml
```

Now feel free to run some test to verify the Static Egress Gateway is working as expected.

Hope it helps!

**References:**

* [Configure Static Egress Gateway in Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/configure-static-egress-gateway)