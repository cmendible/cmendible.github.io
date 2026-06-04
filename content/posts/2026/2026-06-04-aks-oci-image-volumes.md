---
author: Carlos Mendible
categories:
- azure
- kubernetes
date: "2026-06-04"
description: 'AKS: Mount Versioned Content as OCI Image Volumes (No Docker, No Init Containers)'
images: ["/assets/img/posts/aks.png"]
draft: false
tags: ["aks", "containers", "terraform", "oci", "oras"]
title: 'AKS: Mount Versioned Content as OCI Image Volumes'
---

Kubernetes 1.36 graduated **image volumes** to stable. This feature lets the kubelet pull any OCI artifact from a registry and mount its filesystem directly into pods as a **read-only volume** — no init containers, no `emptyDir` copies, no custom CSI drivers. In this post I'll walk through a complete example: packaging versioned project content as a scratch-based OCI image and serving it from a Go HTTP API running on AKS.

> All the code for this post is available at [cmendible/azure.samples — aks_oci_artifact_loader](https://github.com/cmendible/azure.samples/tree/main/aks_oci_artifact_loader).

## Why Image Volumes?

The classic pattern for injecting versioned config or data files into a pod involves either baking them into the application image (couples content to code), using an `emptyDir` + init container to copy files at startup (verbose manifests, extra container lifecycle), or a ConfigMap/Secret (size limits, not suited for binary assets). 

Image volumes give you a cleaner model: **the content is just another OCI image**. You version it, push it to your registry, and reference it by tag or digest. The kubelet pulls and mounts it before any container starts.

```
┌───────────────────────────────────────────────┐
│  Pod                                           │
│                                                │
│  ┌──────────────┐    /content (read-only)      │
│  │   engine     │◄──────────────────────       │
│  │  (Go HTTP)   │                    ▲         │
│  └──────────────┘                    │         │
│                         image volume │         │
└─────────────────────────────────────┼─────────┘
                                       │
                         ┌─────────────┴──────────┐
                         │  ACR                    │
                         │  project-content:v1.0.0 │
                         │  (scratch-based image)  │
                         └────────────────────────┘
```

## Project Structure

```
.
├── engine/               # Go HTTP API server
│   ├── main.go
│   ├── go.mod
│   └── Dockerfile        # multi-stage, distroless final image
├── project-content/      # versioned content bundle
│   ├── config/
│   ├── data/
│   └── rules/
├── oci/
│   └── push-artifact.sh  # packages content as OCI image via ORAS (no Docker)
├── terraform/            # AKS 1.36 + ACR
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── k8s/
│   └── deployment.yaml   # uses volumes[].image (image volumes, k8s 1.36)
└── Makefile
```

## Prerequisites

| Tool | Version |
|------|---------|
| Azure CLI (`az`) | latest |
| ORAS | ≥ 1.2 — auto-installed by `make push-content` |
| Go | ≥ 1.23 (local dev only) |
| Terraform | ≥ 1.7 |
| kubectl | ≥ 1.35 |
| jq | any |

> **No local Docker daemon required.** The engine image is built in ACR Tasks (`az acr build`). The content artifact is pushed with ORAS.

## 1. Provision the Infrastructure (Terraform)

The Terraform configuration creates an AKS 1.36 cluster and an ACR with the kubelet identity granted `AcrPull`.

### terraform/variables.tf

``` terraform
variable "resource_group_name" {
  description = "Name of the Azure resource group."
  type        = string
  default     = "rg-oci-artifact-demo"
}

variable "location" {
  description = "Azure region. Run `az aks get-versions --location <region>` to verify 1.36 availability."
  type        = string
  default     = "swedencentral"
}

variable "acr_name" {
  description = "Globally unique Azure Container Registry name (3-50 alphanumeric characters)."
  type        = string
  # Override with: -var acr_name=myuniqueacr
}

variable "aks_name" {
  description = "AKS cluster name."
  type        = string
  default     = "aks-oci-demo"
}

variable "kubernetes_version" {
  description = "Kubernetes version. Must be >= 1.36 for stable image volumes support."
  type        = string
  default     = "1.36"

  validation {
    condition     = tonumber(split(".", var.kubernetes_version)[1]) >= 36
    error_message = "kubernetes_version must be 1.36 or higher. Image volumes are stable in 1.36."
  }
}

variable "node_count" {
  description = "Number of nodes in the default node pool."
  type        = number
  default     = 2
}

variable "node_vm_size" {
  description = "VM SKU for the default node pool. Run `az aks list-vm-skus --location <region>` to verify availability."
  type        = string
  default     = "Standard_D2s_v3"
}
```

### terraform/main.tf

``` terraform
terraform {
  required_version = ">= 1.7"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}

provider "azurerm" {
  features {}
}

# ── Resource Group ────────────────────────────────────────────────────────────

resource "azurerm_resource_group" "main" {
  name     = var.resource_group_name
  location = var.location
}

# ── Azure Container Registry ──────────────────────────────────────────────────

resource "azurerm_container_registry" "main" {
  name                = var.acr_name
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "Standard"
  admin_enabled       = false
}

# ── AKS Cluster ───────────────────────────────────────────────────────────────

resource "azurerm_kubernetes_cluster" "main" {
  name                = var.aks_name
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = var.aks_name
  kubernetes_version  = var.kubernetes_version

  default_node_pool {
    name       = "system"
    node_count = var.node_count
    vm_size    = var.node_vm_size
    os_sku     = "AzureLinux"
    upgrade_settings {
      max_surge = "1"
    }
  }

  identity {
    type = "SystemAssigned"
  }

  # Enable OIDC issuer so workload identity can pull from ACR if needed
  oidc_issuer_enabled       = true
  workload_identity_enabled = true

  network_profile {
    network_plugin      = "azure"
    network_policy      = "cilium"
    network_plugin_mode = "overlay"
    network_data_plane  = "cilium"
  }
}

# ── ACR pull permission for AKS kubelet identity ──────────────────────────────
# This allows AKS nodes to pull both the engine image and the content image
# from ACR (including image volumes).

resource "azurerm_role_assignment" "aks_acr_pull" {
  principal_id                     = azurerm_kubernetes_cluster.main.kubelet_identity[0].object_id
  role_definition_name             = "AcrPull"
  scope                            = azurerm_container_registry.main.id
  skip_service_principal_aad_check = true
}
```

### Deploy

``` bash
export ACR_NAME=myuniqueacr   # globally unique, 3-50 alphanumeric chars
make tf-apply
# After apply, configure kubectl:
az aks get-credentials --resource-group rg-oci-artifact-demo --name aks-oci-demo
```

## 2. The Engine (Go HTTP Server)

The engine is a simple PoC Go HTTP server that exposes two endpoints:

| Endpoint | Description |
|----------|-------------|
| `GET /tree` | Returns a JSON tree of the mounted `/content` directory |
| `GET /health` | Health check, returns `{"status":"ok"}` |

The content path defaults to `/content` and can be overridden with the `CONTENT_PATH` environment variable.

### engine/main.go

``` go
package main

import (
	"encoding/json"
	"log"
	"net/http"
	"os"
	"path/filepath"
)

type Node struct {
	Name     string  `json:"name"`
	Path     string  `json:"path"`
	IsDir    bool    `json:"is_dir"`
	Children []*Node `json:"children,omitempty"`
}

func buildTree(root, current string) (*Node, error) {
	info, err := os.Stat(current)
	if err != nil {
		return nil, err
	}

	rel, err := filepath.Rel(root, current)
	if err != nil {
		return nil, err
	}

	node := &Node{
		Name:  info.Name(),
		Path:  "/" + filepath.ToSlash(rel),
		IsDir: info.IsDir(),
	}

	if !info.IsDir() {
		return node, nil
	}

	entries, err := os.ReadDir(current)
	if err != nil {
		return nil, err
	}

	for _, entry := range entries {
		child, err := buildTree(root, filepath.Join(current, entry.Name()))
		if err != nil {
			return nil, err
		}
		node.Children = append(node.Children, child)
	}

	return node, nil
}

func treeHandler(w http.ResponseWriter, r *http.Request) {
	contentPath := os.Getenv("CONTENT_PATH")
	if contentPath == "" {
		contentPath = "/content"
	}

	tree, err := buildTree(contentPath, contentPath)
	if err != nil {
		http.Error(w, "failed to read content path: "+err.Error(), http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	if err := json.NewEncoder(w).Encode(tree); err != nil {
		log.Printf("error encoding response: %v", err)
	}
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Write([]byte(`{"status":"ok"}`))
}

func main() {
	http.HandleFunc("/tree", treeHandler)
	http.HandleFunc("/health", healthHandler)

	addr := ":8080"
	log.Printf("engine listening on %s", addr)
	log.Fatal(http.ListenAndServe(addr, nil))
}
```

The engine image is built directly in ACR Tasks — no local Docker daemon needed:

``` bash
make push-engine   # runs: az acr build --registry $ACR_NAME --image engine:latest ./engine
```

## 3. Packaging Content as an OCI Image

This is the most interesting part. You cannot just `oras push` a tarball and expect Kubernetes to mount it. **containerd validates that the `rootfs.diff_ids` in the image config match the SHA256 of each layer's _uncompressed_ content.** A bare `oras push` without a proper OCI image config produces an artifact manifest (no `diff_ids`), which causes:

```
failed to unpack image volume: mismatched image rootfs and manifest layers
```

The script in `oci/push-artifact.sh` handles this correctly:

1. Create an **uncompressed** tar of the content directory.
2. Compute `diff_id = sha256(<uncompressed tar>)`.
3. Compress the tar with gzip for the actual layer blob.
4. Write a valid OCI image config JSON with `rootfs.diff_ids`.
5. Push with `oras push --config config.json:application/vnd.oci.image.config.v1+json`.

### oci/push-artifact.sh

``` bash
#!/usr/bin/env bash
set -euo pipefail

: "${ACR_NAME:?ACR_NAME environment variable is required}"
: "${PROJECT_TAG:=project-content:v1.0.0}"

REGISTRY="${ACR_NAME}.azurecr.io"
CONTENT_DIR="$(cd "$(dirname "$0")/../project-content" && pwd)"

TMP="$(mktemp -d)"
LAYER_TAR_UNCOMPRESSED="${TMP}/layer.tar"
LAYER_TAR_GZ="${TMP}/layer.tar.gz"
CONFIG_JSON="${TMP}/config.json"

cleanup() { rm -rf "${TMP}"; }
trap cleanup EXIT

echo "Authenticating to ${REGISTRY} (no Docker required)..."
TOKEN=$(az acr login --name "${ACR_NAME}" --expose-token \
  --output tsv --query accessToken)
oras login "${REGISTRY}" \
  --username "00000000-0000-0000-0000-000000000000" \
  --password "${TOKEN}"

echo "Creating layer tar from ${CONTENT_DIR}..."
# Step 1: uncompressed tar
tar -cf "${LAYER_TAR_UNCOMPRESSED}" -C "${CONTENT_DIR}" .

# Step 2: diff_id = SHA256 of the UNCOMPRESSED layer
DIFF_ID="sha256:$(sha256sum "${LAYER_TAR_UNCOMPRESSED}" | cut -d' ' -f1)"
echo "  diff_id: ${DIFF_ID}"

# Step 3: compress for storage / transfer efficiency
gzip -9 -c "${LAYER_TAR_UNCOMPRESSED}" > "${LAYER_TAR_GZ}"

# Step 4: write a valid OCI image config with rootfs.diff_ids
# NOTE: architecture is hardcoded to amd64 here. Content layers are
# arch-neutral, but containerd performs a platform check on the image
# config. If you use ARM-based node pools (e.g. Cobalt Dpls_v6),
# change this to "arm64" — or build a multi-platform manifest index.
cat > "${CONFIG_JSON}" <<EOF
{
  "architecture": "amd64",
  "os": "linux",
  "rootfs": {
    "type": "layers",
    "diff_ids": ["${DIFF_ID}"]
  }
}
EOF

echo "Pushing OCI image to ${REGISTRY}/${PROJECT_TAG}..."
oras push "${REGISTRY}/${PROJECT_TAG}" \
  --disable-path-validation \
  --config "${CONFIG_JSON}:application/vnd.oci.image.config.v1+json" \
  "${LAYER_TAR_GZ}:application/vnd.oci.image.layer.v1.tar+gzip"

echo "Image pushed. Digest:"
oras resolve "${REGISTRY}/${PROJECT_TAG}"
echo ""
echo "Pin by digest for production:"
echo "  ${REGISTRY}/project-content@\$(oras resolve ${REGISTRY}/${PROJECT_TAG})"
```

Push the content:

``` bash
make push-content   # auto-installs ORAS if not present, then runs push-artifact.sh
```

For production, **pin by digest** to prevent silent content changes:

``` yaml
reference: <acr>.azurecr.io/project-content@sha256:<digest>
```

## 4. The Kubernetes Deployment

The key piece is `volumes[].image`. The kubelet pulls the OCI image and mounts its filesystem at `mountPath` before any containers start — read-only, no init container required.

### k8s/deployment.yaml

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-engine
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-engine
  template:
    metadata:
      labels:
        app: demo-engine
    spec:
      # Harden the pod: run as non-root, use a restricted security context.
      securityContext:
        runAsNonRoot: true
        runAsUser: 65532
        fsGroup: 65532
      volumes:
        - name: project-content
          image:
            reference: <ACR_LOGIN_SERVER>/project-content:v1.0.0
            pullPolicy: Always   # use IfNotPresent + digest for production

      containers:
        - name: engine
          # Pin engine to an immutable tag or digest in production —
          # 'latest' is used here for brevity but prevents reliable rollback.
          image: <ACR_LOGIN_SERVER>/engine:latest
          ports:
            - containerPort: 8080
          env:
            - name: CONTENT_PATH
              value: /content
          volumeMounts:
            - name: project-content
              mountPath: /content
              readOnly: true
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 128Mi
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 20
---
apiVersion: v1
kind: Service
metadata:
  name: demo-engine
spec:
  selector:
    app: demo-engine
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

Deploy by substituting your ACR login server:

``` bash
make deploy   # sed replaces <ACR_LOGIN_SERVER> then kubectl apply
```

## 5. Smoke Test

``` bash
make smoke-test
```

Expected `/tree` response:

``` json
{
  "name": "content",
  "path": "/",
  "is_dir": true,
  "children": [
    { "name": "config", "path": "/config", "is_dir": true, "children": [...] },
    { "name": "data",   "path": "/data",   "is_dir": true, "children": [...] },
    { "name": "rules",  "path": "/rules",  "is_dir": true, "children": [...] }
  ]
}
```

## 6. How It All Fits Together

When you run `make deploy` and the pod starts, here is what happens:

1. The **kubelet** reads the pod spec and finds `volumes[].image`.
2. It authenticates to ACR using the kubelet managed identity (AcrPull granted by Terraform). **Important:** image-volume pulls go through the node's credential provider (the kubelet MI), *not* pod-level `imagePullSecrets`. If you use a registry that isn't attached to AKS via managed identity, you must configure the credential provider at the node level.
3. containerd pulls the `project-content` OCI image, verifies the layer digest against `rootfs.diff_ids` in the config, and unpacks the layer into an overlay snapshot.
4. The snapshot is bind-mounted read-only at `/content` in the `engine` container.
5. The `engine` container starts and the Go server immediately has access to the versioned files under `/content`.

To roll out new content, push a new tag (or digest), update the manifest reference, and `make deploy`. No application code change, no image rebuild. To **roll back**, simply revert the `reference` to the previous tag or digest and re-apply.

> **Troubleshooting:** if a pod gets stuck on startup, image-volume pull failures appear as pod events *before* any container starts — check `kubectl describe pod <name>` and look for `FailedMount` / `FailedToUnpackImageVolume` events, then inspect kubelet logs on the node.

## Teardown

``` bash
make tf-destroy
```

Hope it helps!

**References:**

* [Kubernetes image volumes](https://kubernetes.io/docs/tasks/configure-pod-container/image-volumes/)
* [ORAS — OCI Registry As Storage](https://oras.land/)
* [az acr build](https://learn.microsoft.com/en-us/cli/azure/acr?view=azure-cli-latest#az-acr-build)
* [Sample code on GitHub](https://github.com/cmendible/azure.samples/tree/main/aks_oci_artifact_loader)
