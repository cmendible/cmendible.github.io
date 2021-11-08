---
author: Carlos Mendible
categories:
- azure
- kubernetes
date: "2021-11-08T10:00:00Z"
description: 'AKS: Open Service Mesh & mTLS'
images: ["/assets/img/posts/osm.png"]
draft: false
tags: ["aks", "osm"]
title: 'AKS: Open Service Mesh & mTLS'
---

[Open Service Mesh (OSM)](https://openservicemesh.io/) is a lightweight and extensible cloud native service mesh, easy to install and configure and with features as mTLS to secure your microservice environments.

Now that [Open Service Mesh (OSM)](https://openservicemesh.io/) integration with Azure Kubernetes Service (AKS) is GA (Check the [announcement](https://techcommunity.microsoft.com/t5/apps-on-azure/open-service-mesh-osm-integration-with-azure-kubernetes-service/ba-p/2902151)) I'll show you not only to deploy it but also how to add your microservices to the mesh so communication between them is encrypted.

## Use terraform to Deploy an AKS cluster with OSM and Monitoring enabled

### Create providers.tf with the following contents:

``` terraform
terraform {
  required_version = "> 0.14"
  required_providers {
    azurerm = {
      version = ">= 2.83.0"
    }
  }
}

provider "azurerm" {
  features {}
}
```

**Note:** be sure to use `azurerm` provide version 2.83.0 or higher.

### Create variables.tf with the following contents:

``` terraform
# Location of the services
variable "location" {
  default = "west europe"
}

# Resource Group Name
variable "resource_group" {
  default = "aks-osm"
}

# Name of the AKS cluster
variable "aks_name" {
  default = "aks-osm"
}
```

### Create main.tf with the following contents:

``` terraform	
# Create Resource Group
resource "azurerm_resource_group" "rg" {
  name     = var.resource_group
  location = var.location
}

resource "azurerm_log_analytics_workspace" "workspace" {
  name                = var.aks_name
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
}

# Create the AKS cluster.
resource "azurerm_kubernetes_cluster" "aks" {
  name                = var.aks_name
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = var.aks_name

  default_node_pool {
    name            = "default"
    node_count      = 3
    vm_size         = "Standard_D2s_v3"
    os_disk_size_gb = 30
    os_disk_type    = "Ephemeral"
  }

  # Using Managed Identity
  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin = "azure"
    network_policy = "calico"
  }

  role_based_access_control {
    enabled = true
  }

  addon_profile {
    kube_dashboard {
      enabled = false
    }
    open_service_mesh {
      enabled = true
    }
    oms_agent {
      enabled                    = true
      log_analytics_workspace_id = azurerm_log_analytics_workspace.workspace.id
    }
  }
}
```

**Note:** `open_service_mesh` and `oms_agent` are enabled.

### Create metrics_configmap.yaml with the following contents:

``` yaml
kind: ConfigMap
apiVersion: v1
data:
  schema-version: v1
  config-version: ver1
  osm-metric-collection-configuration: |-
    # OSM metric collection settings
    [osm_metric_collection_configuration]
      [osm_metric_collection_configuration.settings]
          # Namespaces to monitor
          monitor_namespaces = ["default"]
metadata:
  name: container-azm-ms-osmconfig
  namespace: kube-system
```

**Note:** `monitor_namespaces` is set to `default`, but you can add more namespaces to monitor.

### Deploy the cluster

Runthe following commands:

``` shell
terraform init
terraform plan -out tf.plan
terraform apply ./tf.plan
```

If the deployment fails with the following message:

"OpenServiceMesh addon is not allowed since feature 'Microsoft.ContainerService/AKS-OpenServiceMesh' is not enabled. Please see https://aka.ms/aks/previews for how to enable features."

Make sure you register the `AKS-OpenServiceMesh` feature for your subscription.

``` shell
az feature register --namespace "Microsoft.ContainerService" --name "AKS-OpenServiceMesh"
```

Check if the feature is registered:

``` shell
az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/AKS-OpenServiceMesh')].{Name:name,State:properties.state}"
az provider register --namespace Microsoft.ContainerService
```

Once registered refresh the  `Microsoft.ContainerService` regsitartion:

``` shell
az provider register --namespace Microsoft.ContainerService
```

And once again check the status:

``` shell
az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService')].{Name:name,State:properties.state}"
```

Retry the deployment:

``` shell
terraform apply ./tf.plan
```

## Check OSM status and version:

### Get the cluster credentials:

``` shell
az aks get-credentials -g aks-osm -n aks-osm
```

### Check the status of all OSM components:

``` shell
kubectl get deploy,po,svc -n kube-system --selector app=osm-controller
```

### Check the OSM version:

``` shell
kubectl get deployment -n kube-system osm-controller -o yaml | grep -i image:
```

### Check OSM configuration:

``` shell
kubectl get meshconfig osm-mesh-config -n kube-system -o yaml
```

### Check that Permissive Traffic Policy Mode is enabled by default:

``` shell
kubectl get meshconfig osm-mesh-config -n kube-system -o yaml | grep -i enablePermissiveTrafficPolicyMode
```

## Configure OSM to monitor a namespace

To tell OSM to monitor a namespace, a label must be added to the namespace. In the sample below, the namespace `default` is labled for monitoring. 

``` shell
kubectl label ns default openservicemesh.io/monitored-by=osm
```

Note: The label `openservicemesh.io/monitored-by` does not enables sidecar injection.

If you also want to enable automatic side-car injection for the `default` namespace run:

``` shell
kubectl annotate namespace default openservicemesh.io/sidecar-injection=enabled
```

## Configure OSM to enable metrics for a namespace

To tell OSM to enable metrics for a namespace, a label must be added to the namespace. 

``` shell
kubectl annotate ns default "openservicemesh.io/metrics=enabled"
```

In order for Azure Monitor to read the metrics, deploy the `metrics-configmap.yaml` file to the cluster. 

``` shell
kubectl apply -f ./metrics.configmap.yaml
```

## Add Microservices to the mesh

Run an `nginx` server with the `openservicemesh.io/sidecar-injection=enabled` annotation, so OSM injects the `envoy` sidecar.

``` shell
k run nginx --image nginx --annotations="openservicemesh.io/sidecar-injection=enabled"
k expose po nginx --port 80 --target-port 80
```

Now run a `buybox` pod with the `openservicemesh.io/sidecar-injection=enabled` annotation:

``` shell
kubectl run -it --rm busybox --image busybox --annotations="openservicemesh.io/sidecar-injection=enabled" -- sh
```

Form the shell prompt, run:

``` shell
wget -O- http://nginx
```

> That's it! That request was secured via mTLS

Open another terminal and run: 

``` shell	
kubectl get po
```

Because OSM injected the `envoy` sidecar into each pod, you'll find that both the `nginx` and `buybox` pods have 2 containers.

## Azure Monitor metrics

Run the following Kusto query to get the OSM metrics:

``` shell
InsightsMetrics
| where Name contains "envoy"
| extend t=parse_json(Tags)
```

Hope it helps!!!

Please find the complete sample [here](https://github.com/cmendible/azure.samples/tree/main/aks_osm_mtls)

References:

* [Open Service Mesh AKS add-on](https://docs.microsoft.com/en-us/azure/aks/open-service-mesh-about)
* [Open Service Mesh](https://openservicemesh.io/)
