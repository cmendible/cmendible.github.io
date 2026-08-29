---
author: Carlos Mendible
categories:
- azure
date: "2026-07-29"
description: 'Use the azqr alternative-vm-sku command to instantly find compatible Azure VM SKU alternatives ranked by a multi-factor compatibility score.'
draft: false
images: ["/assets/img/posts/azqr_readme.png"]
tags: ["azqr", "azure", "virtual-machines", "sku"]
title: 'Find Alternative Azure VM SKUs with azqr'
---

Running an Azure VM on an older SKU generation, hitting quota limits, or simply looking for an alternative size? The new `alternative-vm-sku` command in [Azure Quick Review (azqr)](https://github.com/azure/azqr) makes it trivial to discover compatible alternatives for any Azure VM SKU ranked by a multi-factor compatibility score.

> Whether you're modernizing workloads, responding to quota constraints, or optimizing costs, having a ranked list of compatible SKU alternatives removes the guesswork from VM size selection.

## Prerequisites

Before you begin, ensure you have:

- Latest version of [azqr](https://github.com/azure/azqr) installed
- Basic familiarity with Azure VM SKU naming conventions

### Install the Latest azqr

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/azure/azqr/main/scripts/install.sh)"
```

## The alternative-vm-sku Command

The `alternative-vm-sku` command searches azqr's embedded catalog of Azure VM SKUs and returns the top N alternatives for any given SKU, sorted by compatibility score. The result is a JSON document containing the source SKU details and the ranked recommendations.

```bash
azqr alternative-vm-sku --sku <SKU-name> [--top <N>]
```

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `--sku` | `-s` | *(required)* | Azure VM SKU name to find alternatives for |
| `--top` | `-n` | `10` | Number of alternative SKUs to return |

## How the Compatibility Score Works

Each candidate SKU is evaluated against the target using a weighted scoring algorithm (higher is better). Candidates scoring below 0.1 are excluded.

| Factor | Weight | Description |
|--------|--------|-------------|
| vCPU count match | 0.30 | Ratio of the smaller to the larger vCPU count |
| Memory match | 0.25 | Ratio of the smaller to the larger memory (GB) |
| Same base model series | 0.15 | Bonus when candidate shares the same SKU series, ignoring the `_vN` suffix (e.g. `Standard_D16s`) |
| Cross-family bonus | up to +0.10 | Candidates from a **different** family draw from independent capacity pools, making them more useful when the target family is constrained. Base bonus 0.10, reduced by 0.02 per generation behind, floored at 0.0 |
| Same-family penalty | −0.10 | Candidates in the **same** family face the same regional capacity constraints as the target, so they are penalized |
| Version proximity | 0.05 | Full score for same/newer generation; −0.02 per generation behind, floored at 0.0 |
| GPU count match | 0.15 | GPU-enabled SKUs: proportional GPU count match. Non-GPU SKUs: flat 0.10 bonus |
| GPU workload class match | 0.20 | **GPU SKUs only.** Bonus when candidate shares the same N-series workload class: NV (visualization), NC (compute), ND (distributed). Weighted higher than memory because GPU VM sizing is driven by partition count and workload type, not raw RAM |
| Data disk match | 0.05 | Ratio of the smaller to the larger max data disk count |
| Accelerated networking | 0.05 | Full score when candidate supports accelerated networking (or target doesn't require it) |

> The cross-family bonus and same-family penalty replace the previous "same-family bonus" (+0.10). This change ensures that alternatives from independent capacity pools rank higher — which is exactly what you want when responding to quota exhaustion or SKU retirement.

## Examples

### General-Purpose Compute Standard_D4s_v5

```bash
azqr alternative-vm-sku --sku Standard_D4s_v5 --top 3
```

```json
{
  "source": {
    "name": "Standard_D4s_v5",
    "family": "standardDSv5Family",
    "vcpus": 4,
    "memoryGb": 16,
    "gpuCount": 0,
    "maxDataDisks": 8,
    "acceleratedNetworking": true
  },
  "alternatives": [
    {
      "sku": {
        "name": "Standard_D4s_v6",
        "family": "StandardDsv6Family",
        "vcpus": 4,
        "memoryGb": 16,
        "gpuCount": 0,
        "maxDataDisks": 12,
        "acceleratedNetworking": true
      },
      "compatibilityScore": 0.933
    },
    {
      "sku": {
        "name": "Standard_D4s_v7",
        "family": "StandardDsv7Family",
        "vcpus": 4,
        "memoryGb": 16,
        "gpuCount": 0,
        "maxDataDisks": 12,
        "acceleratedNetworking": true
      },
      "compatibilityScore": 0.933
    },
    {
      "sku": {
        "name": "Standard_D4s_v4",
        "family": "standardDSv4Family",
        "vcpus": 4,
        "memoryGb": 16,
        "gpuCount": 0,
        "maxDataDisks": 8,
        "acceleratedNetworking": true
      },
      "compatibilityScore": 0.930
    }
  ]
}
```

The top recommendations for a `Standard_D4s_v5` are the newer `v6` and `v7` siblings — identical vCPUs, memory, and networking, just a newer generation. The slightly lower-scored `v4` is the previous generation of the same series. Note that same-family candidates are now penalized (−0.10) to favour cross-family alternatives; within the D-series family this shows up as a modest score reduction compared to the previous algorithm.

### GPU Workloads Standard_NC6s_v3

```bash
azqr alternative-vm-sku --sku Standard_NC6s_v3 --top 3
```

```json
{
  "source": {
    "name": "Standard_NC6s_v3",
    "family": "standardNCSv3Family",
    "vcpus": 6,
    "memoryGb": 112,
    "gpuCount": 1,
    "maxDataDisks": 12,
    "acceleratedNetworking": true
  },
  "alternatives": [
    {
      "sku": {
        "name": "Standard_NV6s_v2",
        "family": "standardNVSv2Family",
        "vcpus": 6,
        "memoryGb": 112,
        "gpuCount": 1,
        "maxDataDisks": 12,
        "acceleratedNetworking": false
      },
      "compatibilityScore": 0.780
    },
    {
      "sku": {
        "name": "Standard_NC8ads_A10_v4",
        "family": "StandardNCADSA10v4Family",
        "vcpus": 8,
        "memoryGb": 100,
        "gpuCount": 1,
        "maxDataDisks": 8,
        "acceleratedNetworking": true
      },
      "compatibilityScore": 0.732
    },
    {
      "sku": {
        "name": "Standard_NV12s_v3",
        "family": "standardNVSv3Family",
        "vcpus": 12,
        "memoryGb": 112,
        "gpuCount": 1,
        "maxDataDisks": 12,
        "acceleratedNetworking": true
      },
      "compatibilityScore": 0.700
    }
  ]
}
```

For the GPU-enabled `Standard_NC6s_v3`, the algorithm now correctly prioritizes the **GPU workload class** (NC = compute) as a key criterion, surfacing same-class NC candidates ahead of NV (visualization) or ND (distributed) SKUs. Within the same class, GPU count and vCPU/memory ratios break ties.

## Practical Use Cases

- **Quota exhaustion**: A specific SKU family is at quota in your target region quickly find the closest alternative.
- **SKU retirement**: Microsoft is retiring a VM series identify the best upgrade path.
- **Multi-region deployments**: Not all SKUs are available in every region find a compatible alternative available in your target region.

Hope it helps!

**References:**

* [Azure Virtual Machine sizes overview](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/overview)
* [Azure VM SKU families and series](https://learn.microsoft.com/en-us/azure/virtual-machines/vm-naming-conventions)
* [Azure subscription and service limits, quotas, and constraints](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits)
* [Azure Quick Review GitHub repository](https://github.com/azure/azqr)
