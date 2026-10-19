---
author: Carlos Mendible
categories:
- azure
date: "2026-10-19"
description: 'Use the azqr vm-modernization plugin to classify your Azure VMs by workload pattern and turn the result into modernization waves for the v6 and v7 VM series.'
draft: false
images: ["/assets/img/azqr_readme.png"]
tags: ["azqr", "azure", "virtual-machines", "modernization"]
title: 'Classify Your Azure VMs into Modernization Waves with azqr'
---

A list of VM sizes only tells you half the story. An AKS node, an application server and a vendor appliance can all run on an older VM series, but each one needs a very different migration plan. Before picking a target size, you need to know **what each VM does, who manages it and what must be checked before it can move**.

That's why moving from v2-v5 to v6 and v7 isn't a resize exercise. Microsoft Learn describes a Discover → Assess → Plan → Modernize → Validate journey that starts with workload classification, not SKU selection. The new `vm-modernization` plugin in [Azure Quick Review (azqr)](https://github.com/azure/azqr) covers the first three stages: it classifies your VMs using Azure Resource Graph, surfaces readiness signals and suggests target SKUs in an Excel report. In this post I'll run the plugin, read the results and use them to build modernization waves.

> Classify first, then build your waves. The starting VM generation decides the *remediation* effort, but how the workload is deployed, stores state and recovers from losing a single VM decides the *modernization* effort.

## Prerequisites

- Latest version of [azqr](https://github.com/azure/azqr) installed
- Reader access on the subscriptions you want to analyze
- An authenticated Azure identity, using Azure CLI, managed identity or a service principal

## Run the plugin

Run it standalone (it skips the resource scan, so it's fast):

```bash
./azqr vm-modernization
```

Or alongside a regular scan, so the results land in the same report:

```bash
./azqr scan --plugin vm-modernization
```

You can scope it with the usual flags, for example `--subscription-id <sub-id>`. The plugin adds a **VM Modernization** sheet to the Excel report.

## Step 1: VM classification (Discover)

Microsoft Learn groups workloads into seven patterns. The key question is *what gets replaced: a pool, a cluster or a single VM?*

| Pattern | Examples | Typical approach |
|---------|----------|------------------|
| **A. Customer-replaceable compute pools** | AKS node pools, ARO machine sets, Batch pools, stateless VMSS, CI agent pools | Create a replacement pool on the target series, drain the old one |
| **B. Image-based desktop and app hosts** | AVD pooled host pools, Citrix random catalogs | Rebuild the golden image, deploy canary hosts, replace in rings |
| **C. Service-managed compute** | Databricks, Data Explorer, Synapse Spark, PostgreSQL/MySQL flexible server | Select a supported size; the service does the replacement |
| **D. Cluster re-creation** | HDInsight, Azure ML compute clusters | Deploy a replacement cluster, redirect jobs |
| **E. Customer-managed application and infrastructure VMs** | LOB apps, web/app servers, DNS, file servers, persistent desktops | Redeploy from a current image (or in-place upgrade) per VM |
| **F. Stateful and clustered workloads** | Domain controllers, SQL Always On, SAP, Kafka, Cassandra, MongoDB | Add nodes/replicas on the target series, sync, transfer the role |
| **G. Certified ISV virtual appliances** | Palo Alto, Check Point, Fortinet, F5, backup proxies | Vendor certification first, then parallel deployment and traffic shift |

If a workload doesn't match any row, treat it as pattern **E**.

### How azqr classifies each VM

The plugin checks these signals in order and stops at the first match:

1. **C: Service-managed**: Databricks-managed compute.
2. **A: Compute pool**: ARO cluster nodes and AKS node pools.
3. **B: AVD pooled host pool**.
4. **G: Certified ISV appliance**: matched by image publisher/offer vendor.
5. **A: Compute pool**: any other VMSS member.
6. **E: Customer-managed VM**: the default for everything else.

The output also includes a **Pool Membership** column (`AKS Node Pool`, `ARO Cluster`, `VMSS`, `AVD Pooled Host Pool`, `Azure Databricks`, `Standalone` or the appliance vendor name).

A generic VMSS member shows `Manual Review Required` as its execution method until you confirm it's really a stateless pool.

> Patterns **F** (stateful/clustered, including SAP) and **D** are not detected automatically. Review your `E` rows for domain controllers, clusters, databases and SAP before planning waves.

## Step 2: Readiness signals (Assess)

Now check what has to change before the workload can run on the target series. The report includes:

- **Hyper-V Generation**: the VM's boot generation, not its size series. v6/v7 require Generation 2 (UEFI), so a Generation 1 VM needs an upgrade or a replacement built from a Generation 2 image.
- **Disk Controller Type**: SCSI or NVMe. Moving to NVMe means the OS, drivers and any scripts that reference disk paths must support it.
- **Series Generation**: whether the size is already v6/v7.
- **Azure Disk Encryption**: whether ADE is enabled on any attached disk. Check this before choosing a migration method; ADE-encrypted Linux VMs don't support NVMe, for example.
- **Temp Disk Dependency**: a reminder to check if the app uses the local temporary disk (scratch files, SQL Server `tempdb`). The plugin can't tell from the resource configuration, so you have to. Only `d`-suffixed v6/v7 sizes have a local disk, and it's presented raw as NVMe, so pick a `d` size or move that data to a managed disk.
- **SAP/ISV Certification Gate**: `Manual Review Required` for every pattern except pool- or service-managed A and C (and generic VMSS members, which also require review). The plugin can't rule out SAP or an ISV-certified workload, so confirm support for the target VM family yourself.

> `Manual Review Required` means the plugin didn't verify that requirement, not that the VM is incompatible. A firewall appliance with a suggested size still needs vendor confirmation for it.

## Step 3: Execution method and target SKU (Plan)

The report suggests how to move each workload and which size to consider. For patterns E and G, start with the execution method:

| Situation | Recommendation and what it means |
|-----------|----------------------------------|
| Already on v6/v7 | Nothing to do. |
| Generation 2, older series | **Redeploy**: create a replacement VM from a current, compatible image, then move the application and its data. Review disk compatibility, including the SCSI-to-NVMe guidance. |
| Generation 1, older series | **Manual Review Required**: rebuild from a Generation 2 image or use a supported Generation 1 to Generation 2 / Trusted Launch upgrade. A size change alone isn't enough, and Gen 1 Dv2/Dv3 VMs can't be upgraded in place, so rebuild those. |
| Pools, AVD pooled hosts or service-managed compute (A, B, C) | Change it through the owning pool or service, not VM by VM. For example, replace an AKS node pool using the AKS process. |

These are starting points, not approval to migrate. **Redeploy** doesn't remove the manual checks from Step 2: you still need to confirm temp-disk requirements and any SAP or vendor certification.

The **target SKU** comes from the same embedded capability catalog behind [`alternative-vm-sku`](/2026/07/29/find-alternative-azure-vm-skus-with-azqr/), and its score measures hardware similarity only (vCPU, memory and features). It's a candidate to validate, not a readiness verdict: it doesn't check CPU architecture, temp-disk needs, attached disks or NICs, vendor certification, image readiness, region or zone availability, quota or capacity.

The report also highlights upcoming retirements and price changes, but only for the series in the plugin's embedded dataset (D, Dv2, Dv3, Ev3, A, B, F, G, L and a few HPC/NP series). For anything else, `N/A` means no published lifecycle risk was encoded.

For costs, the plugin uses the Azure Retail Prices API to estimate the **current SKU's** monthly pay-as-you-go cost in USD (hourly price × 730), then applies price changes that haven't taken effect yet, since the current retail rate already includes earlier ones. **Projected Monthly Cost After Increase (USD)** and **Monthly Cost Impact (USD)** describe that increase on the current SKU; they are not the target SKU's price or migration savings. If nothing applies, the impact is zero. Stopped VMs that are still allocated are billed; deallocated VMs show `N/A (VM deallocated)`. Treat all of it as retail estimates, not your final bill.

## From classification to waves

The plugin gives you the inventory and classification; the waves are up to you. Microsoft Learn's sequencing guidance is:

1. **Group waves by application or service dependency**, so related tiers move together.
2. **Don't split tightly coupled tiers** across old and new families for long.
3. **Move non-production before production** for every application.
4. **Keep waves small enough to validate in one window.**

To apply this, sort and filter the sheet by `Workload Pattern`, `Pool Membership` and `Resource Group`. The sheet doesn't export tags, so use a separate inventory if you group by application or environment tags.

| Wave | What goes in | Why |
|------|--------------|-----|
| **Wave 0: Pilot** | One representative pattern-E VM, ideally non-production and Gen 2 | Proves the pattern the later waves repeat |
| **Pool track (A, B, C)** | AKS/VMSS/AVD/Databricks rows | Not per-VM work: replace the pool or cluster with your service's process |
| **Early waves** | Pattern E, non-production, Gen 2 / NVMe-ready, no temp-disk dependency | Lowest remediation effort, builds confidence |
| **Middle waves** | Pattern E, production, grouped by application tier; Gen 1 VMs after the image is remediated | Redeploy keeps the source VM running until validation |
| **Late waves** | Pattern F (stateful/clustered) | Add nodes, let sync finish, transfer roles. Never modernize all quorum members in one wave |
| **Gated waves** | Pattern G (appliances) | Don't start until the vendor confirms the exact target family in writing |

For **E, F and G**, follow the Microsoft Learn per-VM runbook: pilot, execute, validate against the wave exit criteria, and keep the old VMs (or restore points for in-place upgrades) until the rollback window closes. For **A, B, C and manually identified D**, use the owning service's pool, host-pool or cluster procedure. For B, assess the golden image first.

### Redeploy or in-place?

Redeploy from a current Gen 2, NVMe- and MANA-ready image is the default: rollback is just keeping the old VM. The in-place upgrade (SCSI-to-NVMe conversion, plus the Gen 1 to Trusted Launch upgrade for eligible sources) is for when you can't practically rebuild. The conservative route uses a source VM without a temp disk or Azure Disk Encryption, with an OS that is already NVMe-ready. Check the current conversion and Trusted Launch prerequisites; the report signals aren't a full eligibility test.

The platform operations are supported, but the community SCSI-to-NVMe automation script isn't an official Microsoft tool. Test it in non-production, with a backup and a rollback plan, before even considering production.

## Scope and limitations

- The plugin covers **Discover, Assess and Plan** only. Validate is on you.
- ARO nodes are recognized by cluster ownership only, so they aren't split into workers and control plane. Hosted-control-plane ARO and self-managed OpenShift aren't covered. Use supported ARO procedures, contact support before control-plane changes and never stop or deallocate cluster VMs directly.
- Failed lookups are logged as warnings, so missing permissions can leave a VM with the default `E` classification. Double-check unexpected `E` rows.
- The temp-disk gate is `Manual Review Required` for B, E and G (and generic VMSS members); the certification gate for everything except A and C. Resolve them, plus any stateful requirements, before acting on a Redeploy recommendation.

Hope it helps!

**References:**

* [Modernize to the v6 and v7 VM series with a wave-based runbook](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/lifecycle/sizes-v6-v7-modernization-modernize)
* [Discover modernization pattern by workload type](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/lifecycle/sizes-v6-v7-modernization-discover)
* [Plan your v6 and v7 modernization (wave sequencing)](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/lifecycle/sizes-v6-v7-modernization-plan#wave-sequencing)
* [Assess readiness for the v6 and v7 series](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/lifecycle/sizes-v6-v7-modernization-assess)
* [azqr internal plugins documentation](https://azure.github.io/azqr/docs/plugins/internal-plugins/)
* [Azure Quick Review on GitHub](https://github.com/Azure/azqr)
* [Convert a VM from SCSI to NVMe in place](https://learn.microsoft.com/en-us/azure/virtual-machines/migration/scsi-to-nvme-migration)
