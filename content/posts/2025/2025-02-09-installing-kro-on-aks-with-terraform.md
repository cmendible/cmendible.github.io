---
author: Carlos Mendible
categories:
- azure
- kubernetes
date: "2025-02-09"
description: 'Installing kro on AKS with Terraform'
images: ["/assets/img/posts/kubernetes.png"]
draft: false
tags: ["kro", "terraform"]
title: 'Installing kro on AKS with Terraform'
---

In this post, we will go through the steps required to install Kube Resource Orchestrator (`kro`) on an Azure Kubernetes Service (AKS) cluster using Terraform. `kro` is an open-source project that enables you to define custom Kubernetes APIs using simple and straightforward configuration.

> Defining custom Kubernetes APIs is becoming essential to simplify the developer experience and increase productivity.

By creating custom APIs you enable developers to create a group of Kubernetes objects and the logical operations between them using a simple comfiguration file. This approach simplifies the deployment of complex applications and reduces the risk of human errors.

To install `kro` on an AKS cluster, follow these steps:

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
  default = "rg-kro-demo"
}

variable "location" {
  default = "spaincentral"
}

variable "cluster_name" {
  default = "aks-kro"
}

variable "dns_prefix" {
  default = "aks-kro"
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
  name                = "vnet-aks-kro"
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
```

> The sample requires an ingress controller to be deployed in the cluster. The ingress controller is used to route traffic to the web application. That's why we enable the `web_app_routing` add-on.

## Kube Resource Orchestrator (kro)

Install Kube Resource Orchestrator (`kro`) using the following Terraform configuration:

```bash
# Install the kro helm chart
resource "helm_release" "kro" {
  create_namespace = true
  name             = "kro"
  chart            = "kro"
  version          = "0.2.1"
  namespace        = "kro"
  repository       = "oci://ghcr.io/kro-run/kro"
}
```

> At the time of writing, the latest version of the kro chart is 0.2.1.

## Apply the Terraform configuration

Apply the Terraform configuration using the following commands:

```bash
terraform init
terraform apply
```

## Deploy a ResourceGraphDefinition

A `ResourceGraphDefinition` allows you to define a custom Kubernetes API and group of Kubernetes objects and the logical operations between them, effectively defining a complete Application Stack.

In this case we will create the definition for a simple web application. This Application Stack will includes a deployment, a service, and an ingress.

Create a `ResourceGraphDefinition` using the following command:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: kro.run/v1alpha1
kind: ResourceGraphDefinition
metadata:
  name: simple-web-app
spec:
  # kro uses this simple schema to create your CRD schema and apply it
  # The schema defines what users can provide when they instantiate the RGD (create an instance).
  schema:
    apiVersion: v1alpha1
    kind: SimpleWebApplication # This is the kind that defines the Application Stack
    spec:
      # Spec fields that users can provide.
      name: string
      image: string | default="nginx"
      ingress:
        enabled: boolean | default=false
    status:
      # Fields the controller will inject into instances status.
      deploymentConditions: ${deployment.status.conditions}
      availableReplicas: ${deployment.status.availableReplicas}

  # Define the resources this API will manage.
  resources:
    - id: deployment
      template:
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: ${schema.spec.name} # Use the name provided by user
        spec:
          replicas: 3
          selector:
            matchLabels:
              app: ${schema.spec.name}
          template:
            metadata:
              labels:
                app: ${schema.spec.name}
            spec:
              containers:
                - name: ${schema.spec.name}
                  image: ${schema.spec.image} # Use the image provided by user
                  ports:
                    - containerPort: 80

    - id: service
      template:
        apiVersion: v1
        kind: Service
        metadata:
          name: ${schema.spec.name}-service
        spec:
          selector: ${deployment.spec.selector.matchLabels} # Use the deployment selector
          ports:
            - protocol: TCP
              port: 80
              targetPort: 80

    - id: ingress
      includeWhen:
        - ${schema.spec.ingress.enabled} # Only include if the user wants to create an Ingress
      template:
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        metadata:
          name: ${schema.spec.name}-ingress
        spec:
          # Remember we are using the webapp routing add-on
          ingressClassName: webapprouting.kubernetes.azure.com 
          rules:
          - http:
              paths:
              - backend:
                  service:
                    name: ${service.metadata.name} # Use the service name
                    port:
                      number: 80
                path: /
                pathType: Prefix
EOF
```

## Create the web application

Now you can create a simple web application using the following command:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: kro.run/v1alpha1
kind: SimpleWebApplication
metadata:
  name: test-httpd
spec:
  name: test-httpd
  image: httpd
  ingress:
    enabled: true
EOF
```

> By creating `ResourceGraphDefinition` objects you are effectively simplifying the deployment of complex applications and abstracting the complexity of the Kubernetes objects.

> If you have Azure Service Operator (`aso`) installed in your cluster you can add Azure services as part of your `ResourceGraphDefinition` extending the capabilities of your Application Stack beyond the Kubernetes cluster.

Once the web application is created, you can access it trough the public IP of the ingress service:

```bash
IP_ADDRESS=$(kubectl get service -n app-routing-system nginx -o jsonpath="{.status.loadBalancer.ingress[0].ip}")
curl http://$IP_ADDRESS
```

Hope it helps!

**References:**

* [kro (Kube Resource Orchestrator)](https://kro.run/docs/overview)
* [Installing Azure Service Operator on AKS with Terraform](https://carlos.mendible.com/2025/02/02/installing-azure-service-operator-on-aks-with-terraform/)
