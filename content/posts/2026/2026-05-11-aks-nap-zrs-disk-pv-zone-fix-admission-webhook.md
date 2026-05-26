---
author: Carlos Mendible
categories:
- azure
- kubernetes
date: "2026-05-11"
description: 'AKS NAP: ZRS Disk PV Zone Fix with a Mutating Admission Webhook'
images: ["/assets/img/posts/aks.png"]
draft: false
tags: ["aks", "containers", "terraform", "karpenter", "nap"]
title: 'AKS NAP: ZRS Disk PV Zone Fix with a Mutating Admission Webhook'
---

If you are running AKS with Node Auto Provisioning (NAP/Karpenter) and using Azure Disk ZRS (Zone-Redundant Storage) Persistent Volumes with `volumeBindingMode: Immediate`, you may have noticed that pods get stuck in `Pending` state. In this post, I'll show you a **temporary workaround** using a Kubernetes mutating admission webhook that fixes this scheduling issue.

> **⚠️ Important:** This is a **temporary workaround / Proof of Concept**. The root cause is tracked in [kubernetes-sigs/karpenter#2743](https://github.com/kubernetes-sigs/karpenter/pull/2743). Once the fix lands in Karpenter and is rolled out to AKS NAP, this webhook will no longer be needed.
>
> **Update (2026-05-15):** PR [#2743](https://github.com/kubernetes-sigs/karpenter/pull/2743) has already been merged upstream in Karpenter, but it is **not part of AKS NAP yet**. Until AKS NAP picks up a release containing that change, this webhook workaround is still an option.
>
> **Update (2026-05-26):** The fix has been released in [karpenter-provider-azure v1.12.1](https://github.com/Azure/karpenter-provider-azure/releases/tag/v1.12.1). Once your AKS NAP is running v1.12.1 or later, this webhook workaround will no longer be needed.

## The Problem

When a ZRS Azure Disk PV is provisioned with `volumeBindingMode: Immediate`, the Azure Disk CSI driver writes the PV `nodeAffinity` as **one `NodeSelectorTerm` per availability zone** (OR semantics across terms):

``` yaml
nodeAffinity:
  required:
    nodeSelectorTerms:
    - matchExpressions:
      - key: topology.disk.csi.azure.com/zone
        operator: In
        values: [eastus2-1]    # term 0 – only zone 1
    - matchExpressions:
      - key: topology.disk.csi.azure.com/zone
        operator: In
        values: [eastus2-2]    # term 1 – only zone 2
    - matchExpressions:
      - key: topology.disk.csi.azure.com/zone
        operator: In
        values: [eastus2-3]    # term 2 – only zone 3
```

Karpenter/NAP only evaluates the **first** `NodeSelectorTerm` when computing where to provision a new node. If a pod's `nodeSelector` requires a different zone (e.g. zone 2), Karpenter provisions a node in zone 1, the scheduler sees an affinity mismatch, and the pod stays `Pending` indefinitely.

## The Fix (Temporary Workaround)

The webhook intercepts PV `CREATE` and `UPDATE` events, detects ZRS disks (via `skuName` containing `ZRS` in the CSI volume attributes), and **merges** all zone values from separate `NodeSelectorTerms` into a single term:

``` yaml
nodeAffinity:
  required:
    nodeSelectorTerms:
    - matchExpressions:
      - key: topology.disk.csi.azure.com/zone
        operator: In
        values: [eastus2-1, eastus2-2, eastus2-3]   # merged – any zone OK
```

Karpenter now sees a single term with all zones and correctly honours the pod's zone preference when provisioning a node.

## Prerequisites

| Tool | Purpose |
|------|---------|
| Go 1.23+ | Build the webhook binary |
| Azure CLI (`az`) | ACR build, AKS credentials |
| Terraform ≥ 1.5 | Provision AKS + ACR |
| kubectl | Deploy and inspect resources |
| cert-manager v1.15+ | TLS certificate management (deployed by Terraform) |

## 1. Provision the AKS NAP cluster

### Create the providers.tf file

Create a file called `providers.tf` with the following contents:

``` terraform
terraform {
  required_version = ">= 1.8"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.14"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.31"
    }
  }
}

provider "azurerm" {
  features {}
}

provider "helm" {
  kubernetes {
    host                   = azurerm_kubernetes_cluster.main.kube_config[0].host
    client_certificate     = base64decode(azurerm_kubernetes_cluster.main.kube_config[0].client_certificate)
    client_key             = base64decode(azurerm_kubernetes_cluster.main.kube_config[0].client_key)
    cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.main.kube_config[0].cluster_ca_certificate)
  }
}

provider "kubernetes" {
  host                   = azurerm_kubernetes_cluster.main.kube_config[0].host
  client_certificate     = base64decode(azurerm_kubernetes_cluster.main.kube_config[0].client_certificate)
  client_key             = base64decode(azurerm_kubernetes_cluster.main.kube_config[0].client_key)
  cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.main.kube_config[0].cluster_ca_certificate)
}
```

### Create the variables.tf file

Create a file called `variables.tf` with the following contents:

``` terraform
variable "resource_group_name" {
  description = "Name of the Azure resource group."
  type        = string
  default     = "rg-aks-nap-webhook"
}

variable "location" {
  description = "Azure region for all resources."
  type        = string
  default     = "eastus2"
}

variable "cluster_name" {
  description = "Name of the AKS cluster."
  type        = string
  default     = "aks-nap-webhook"
}

variable "kubernetes_version" {
  description = "Kubernetes version for the AKS cluster."
  type        = string
  default     = "1.33"
}

variable "system_node_pool_vm_size" {
  description = "VM size for the system node pool."
  type        = string
  default     = "Standard_D4ds_v5"
}

variable "system_node_pool_count" {
  description = "Initial node count for the system node pool."
  type        = number
  default     = 2
}

variable "acr_name" {
  description = "Globally unique name for the Azure Container Registry."
  type        = string
  default     = "acrnapwebhook"
}

variable "cert_manager_chart_version" {
  description = "Version of the cert-manager Helm chart to install."
  type        = string
  default     = "v1.16.3"
}

variable "tags" {
  description = "Tags applied to all resources."
  type        = map(string)
  default = {
    environment = "dev"
    project     = "aks-nap-admission-webhook"
  }
}
```

### Create the main.tf file

Create a file called `main.tf` with the following contents:

``` terraform
# Resource Group
resource "azurerm_resource_group" "main" {
  name     = var.resource_group_name
  location = var.location
  tags     = var.tags
}

# AKS Cluster with Node Auto Provisioning (NAP)
resource "azurerm_kubernetes_cluster" "main" {
  name                = var.cluster_name
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = var.cluster_name
  kubernetes_version  = var.kubernetes_version

  default_node_pool {
    name                         = "system"
    vm_size                      = var.system_node_pool_vm_size
    node_count                   = var.system_node_pool_count
    only_critical_addons_enabled = true

    upgrade_settings {
      max_surge = "33%"
    }
  }

  # Enable Node Auto Provisioning.
  node_provisioning_profile {
    mode = "Auto"
  }

  identity {
    type = "SystemAssigned"
  }

  # Azure CNI + Cilium data plane (required for NAP).
  network_profile {
    network_plugin     = "azure"
    network_policy     = "cilium"
    network_data_plane = "cilium"
    load_balancer_sku  = "standard"
  }

  oidc_issuer_enabled       = true
  workload_identity_enabled = true

  tags = var.tags
}

# Azure Container Registry
resource "azurerm_container_registry" "main" {
  name                = var.acr_name
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "Standard"
  admin_enabled       = false
  tags                = var.tags
}

# Grant the AKS kubelet identity AcrPull
resource "azurerm_role_assignment" "aks_acr_pull" {
  scope                = azurerm_container_registry.main.id
  role_definition_name = "AcrPull"
  principal_id         = azurerm_kubernetes_cluster.main.kubelet_identity[0].object_id
}

# cert-manager (via Helm)
resource "helm_release" "cert_manager" {
  name             = "cert-manager"
  repository       = "https://charts.jetstack.io"
  chart            = "cert-manager"
  version          = var.cert_manager_chart_version
  namespace        = "cert-manager"
  create_namespace = true
  atomic           = true
  cleanup_on_fail  = true
  timeout          = 300

  set {
    name  = "crds.enabled"
    value = "true"
  }

  set {
    name  = "global.leaderElection.namespace"
    value = "cert-manager"
  }

  depends_on = [azurerm_kubernetes_cluster.main]
}
```

### Create the outputs.tf file

Create a file called `outputs.tf` with the following contents:

``` terraform
output "resource_group_name" {
  description = "Name of the resource group containing the AKS cluster."
  value       = azurerm_resource_group.main.name
}

output "cluster_name" {
  description = "Name of the AKS cluster."
  value       = azurerm_kubernetes_cluster.main.name
}

output "get_credentials_command" {
  description = "az CLI command to fetch kubeconfig for this cluster."
  value       = "az aks get-credentials --resource-group ${azurerm_resource_group.main.name} --name ${azurerm_kubernetes_cluster.main.name} --overwrite-existing"
}

output "acr_login_server" {
  description = "Login server hostname for the container registry."
  value       = azurerm_container_registry.main.login_server
}
```

### Deploy the infrastructure

Run the following commands to provision the AKS cluster:

``` bash
cd terraform
export ARM_SUBSCRIPTION_ID=<your-subscription-id>
terraform init
terraform apply
```

After apply, get credentials:

``` bash
$(terraform output -raw get_credentials_command)
```

## 2. The Webhook Code

The webhook is a simple Go application that listens for PV admission reviews and merges zone `NodeSelectorTerms` for ZRS disks.

### Entry point (cmd/webhook/main.go)

Create a file called `cmd/webhook/main.go` with the following contents:

``` go
package main

import (
	"log"
	"net/http"
	"os"

	"github.com/azure-samples/aks-nap-admission-webhook/internal/handler"
)

func main() {
	log.SetOutput(os.Stdout)
	log.SetFlags(log.LstdFlags | log.Lmicroseconds)

	log.Println("[startup] PV Zone Fix Webhook starting on :8443")

	http.HandleFunc("/mutate", handler.HandleMutate)
	http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	})

	if err := http.ListenAndServeTLS(":8443", "/tls/tls.crt", "/tls/tls.key", nil); err != nil {
		log.Fatalf("[fatal] Failed to start HTTPS server: %v", err)
	}
}
```

### Mutation logic (internal/handler/handler.go)

Create a file called `internal/handler/handler.go` with the following contents:

``` go
package handler

import (
	"encoding/json"
	"io"
	"log"
	"net/http"
	"os"
	"sort"
	"strings"
	"time"

	admissionv1 "k8s.io/api/admission/v1"
	corev1 "k8s.io/api/core/v1"
)

// The Azure Disk CSI driver sets nodeAffinity using its own topology key.
const topologyZoneKey = "topology.disk.csi.azure.com/zone"

func HandleMutate(w http.ResponseWriter, r *http.Request) {
	start := time.Now()
	log.Println("[request] Incoming admission review")

	body, err := io.ReadAll(r.Body)
	if err != nil {
		log.Printf("[error] Failed reading request body: %v", err)
		http.Error(w, "cannot read body", http.StatusBadRequest)
		return
	}

	var review admissionv1.AdmissionReview
	if err := json.Unmarshal(body, &review); err != nil {
		log.Printf("[error] Failed to unmarshal AdmissionReview: %v", err)
		http.Error(w, "bad request", http.StatusBadRequest)
		return
	}

	var pv corev1.PersistentVolume
	if err := json.Unmarshal(review.Request.Object.Raw, &pv); err != nil {
		log.Printf("[error] Failed to unmarshal PV: %v", err)
		http.Error(w, "bad PV object", http.StatusBadRequest)
		return
	}

	log.Printf("[pv] Name=%s", pv.Name)

	// 1. Detect if PV is a ZRS Azure Disk (CSI driver)
	isZRS := false
	if pv.Spec.CSI != nil {
		sku := pv.Spec.CSI.VolumeAttributes["skuName"]
		log.Printf("[pv] CSI driver=%s skuName=%s", pv.Spec.CSI.Driver, sku)
		if strings.Contains(sku, "ZRS") {
			isZRS = true
		}
	}

	if !isZRS {
		log.Println("[skip] PV is not ZRS → no mutation applied")
		review.Response = &admissionv1.AdmissionResponse{
			UID:     review.Request.UID,
			Allowed: true,
		}
		writeResponse(w, review, start)
		return
	}

	log.Println("[zrs] ZRS disk detected → evaluating node affinity topology")

	// 2. Extract all zone values from NodeAffinity
	if pv.Spec.NodeAffinity == nil || pv.Spec.NodeAffinity.Required == nil {
		log.Println("[skip] No nodeAffinity found → nothing to merge")
		review.Response = &admissionv1.AdmissionResponse{
			UID:     review.Request.UID,
			Allowed: true,
		}
		writeResponse(w, review, start)
		return
	}

	zones := map[string]struct{}{}
	for _, term := range pv.Spec.NodeAffinity.Required.NodeSelectorTerms {
		for _, expr := range term.MatchExpressions {
			if expr.Key == topologyZoneKey {
				for _, v := range expr.Values {
					if v == "" {
						continue
					}
					log.Printf("[zone] Found zone=%s", v)
					zones[v] = struct{}{}
				}
			}
		}
	}

	if len(zones) == 0 {
		log.Println("[skip] No zones found in nodeAffinity → nothing to merge")
		review.Response = &admissionv1.AdmissionResponse{
			UID:     review.Request.UID,
			Allowed: true,
		}
		writeResponse(w, review, start)
		return
	}

	// 3. Merge zones into a single NodeSelectorTerm
	merged := make([]string, 0, len(zones))
	for z := range zones {
		merged = append(merged, z)
	}
	sort.Strings(merged)

	log.Printf("[merge] Merged zones: %v", merged)

	mergedTerms := []corev1.NodeSelectorTerm{
		{
			MatchExpressions: []corev1.NodeSelectorRequirement{
				{
					Key:      topologyZoneKey,
					Operator: corev1.NodeSelectorOpIn,
					Values:   merged,
				},
			},
		},
	}

	// 4. Build JSONPatch
	patch := []map[string]interface{}{
		{
			"op":    "replace",
			"path":  "/spec/nodeAffinity/required/nodeSelectorTerms",
			"value": mergedTerms,
		},
	}

	patchBytes, _ := json.Marshal(patch)
	log.Printf("[patch] JSONPatch=%s", string(patchBytes))

	// 5. DRY RUN MODE
	if os.Getenv("DRY_RUN") == "true" {
		log.Println("[dry-run] DRY_RUN=true → patch NOT applied")
		review.Response = &admissionv1.AdmissionResponse{
			UID:     review.Request.UID,
			Allowed: true,
		}
		writeResponse(w, review, start)
		return
	}

	// 6. Return the patch
	pt := admissionv1.PatchTypeJSONPatch
	review.Response = &admissionv1.AdmissionResponse{
		UID:       review.Request.UID,
		Allowed:   true,
		Patch:     patchBytes,
		PatchType: &pt,
	}

	writeResponse(w, review, start)
}

func writeResponse(w http.ResponseWriter, review admissionv1.AdmissionReview, start time.Time) {
	resp, err := json.Marshal(review)
	if err != nil {
		log.Printf("[error] Failed to marshal response: %v", err)
		http.Error(w, "cannot marshal response", http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	w.Write(resp)
	log.Printf("[response] Completed in %s", time.Since(start))
}
```

### Dockerfile

Create a `Dockerfile` with the following contents:

``` dockerfile
FROM golang:1.23-alpine AS builder

WORKDIR /build

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -trimpath -ldflags="-s -w" -o /webhook ./cmd/webhook

FROM gcr.io/distroless/static:nonroot

COPY --from=builder /webhook /webhook

USER nonroot:nonroot

ENTRYPOINT ["/webhook"]
```

## 3. Build and push the webhook image

Build inside ACR (no local Docker daemon needed):

``` bash
make acr-build ACR_NAME=<your-acr-name>
```

## 4. Deploy the webhook

Update the image reference in the deployment manifest below (search for `acrnapwebhook.azurecr.io`) to match your ACR, then apply:

``` bash
kubectl apply -f deploy/webhook.yaml
```

The `webhook.yaml` manifest creates all the required resources in namespace `pv-zone-fix-webhook`:

- `Namespace` / `ServiceAccount` / `ClusterRole` / `ClusterRoleBinding`
- `Deployment` (distroless, read-only root FS, runs as non-root)
- `Service` (port 443 → 8443)
- cert-manager `ClusterIssuer` → CA `Certificate` → `Issuer` → TLS `Certificate`
- `MutatingWebhookConfiguration` (CA bundle injected automatically by cert-manager)

> The webhook uses `failurePolicy: Ignore` so it won't block PV creation if the webhook is unavailable.

Verify the pods are ready:

``` bash
kubectl get pods -n pv-zone-fix-webhook
```

## 5. Watch webhook logs

``` bash
kubectl logs -n pv-zone-fix-webhook \
  -l app.kubernetes.io/name=pv-zone-fix-webhook -f
```

Expected output when the webhook merges a PV:

```
[request] Incoming admission review
[pv] Name=pvc-xxxxxxxx-...
[pv] CSI driver=disk.csi.azure.com skuName=Premium_ZRS
[zrs] ZRS disk detected → evaluating node affinity topology
[zone] Found zone=eastus2-1
[zone] Found zone=eastus2-2
[zone] Found zone=eastus2-3
[merge] Merged zones: [eastus2-1 eastus2-2 eastus2-3]
[patch] JSONPatch=[{"op":"replace","path":"/spec/nodeAffinity/required/nodeSelectorTerms","value":[{"matchExpressions":[{"key":"topology.disk.csi.azure.com/zone","operator":"In","values":["eastus2-1","eastus2-2","eastus2-3"]}]}]}]
[response] Completed in 450µs
```

## 6. Test the fix

Create a ZRS StorageClass with `Immediate` binding and a pod pinned to a specific zone:

``` bash
# After applying the test suite, wait for PVC to bind (Immediate binding)
kubectl get pvc -n st-zrs-zone-pin st-pvc-zrs -w

# Inspect the PV nodeAffinity — should be ONE term with all zones merged:
kubectl get pv \
  $(kubectl get pvc -n st-zrs-zone-pin st-pvc-zrs -o jsonpath='{.spec.volumeName}') \
  -o jsonpath='{.spec.nodeAffinity}' | jq .

# Pod is pinned to ZONE-2; verify it lands there:
kubectl get pod -n st-zrs-zone-pin st-pod-zrs-pin -o wide
```

**Without the webhook:** pod stays `Pending` with `incompatible volume requirements ... topology.disk.csi.azure.com/zone In [eastus2-1] not in ... In [eastus2-2]`.

**With the webhook:** pod reaches `Running` in the correct zone.

## Clean up

``` bash
# Delete the webhook
kubectl delete -f deploy/webhook.yaml

# Destroy the infrastructure
cd terraform
terraform destroy
```

Hope it helps!

> **Note:** The fix for this issue has been released in [karpenter-provider-azure v1.12.1](https://github.com/Azure/karpenter-provider-azure/releases/tag/v1.12.1). If your AKS NAP is running v1.12.1 or later, you can remove this webhook workaround.

**References:**

* [Node Autoprovisioning](https://learn.microsoft.com/en-us/azure/aks/node-autoprovision?tabs=azure-cli)
* [kubernetes-sigs/karpenter#2743](https://github.com/kubernetes-sigs/karpenter/pull/2743)
* [karpenter-provider-azure storage e2e tests](https://github.com/Azure/karpenter-provider-azure/blob/6b393df8311e497993b55afde9d08570cd0e2798/test/suites/storage/suite_test.go)
* [Sample code on GitHub](https://github.com/cmendible/azure.samples/tree/main/aks_nap_admission_webhook)
