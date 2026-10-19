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

> Classify first, then build your waves. The starting Hyper-V generation (Gen 1 or Gen 2, not the size series) decides the *remediation* effort, but how the workload is deployed, stores state and recovers from losing a single VM decides the *modernization* effort.

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

If a workload doesn't match any row, treat it as pattern **E**. Persistent desktops (AVD personal host pools) are **E**, not B: a user is bound to a VM, so you can't drain and replace it.

### How azqr classifies each VM

The plugin checks these signals in order and stops at the first match:

1. **C: Service-managed**: Databricks-managed compute (matched on the `Vendor=Databricks` tag).
2. **A: Compute pool**: ARO cluster nodes and AKS node pools.
3. **B: AVD pooled host pool**.
4. **G: Certified ISV appliance**: matched by image publisher/offer vendor.
5. **A: Compute pool**: any other VMSS member.
6. **E: Customer-managed VM**: the default for everything else, including AVD personal host pools.

A **Pool Membership** column shows the match (`AKS Node Pool`, `ARO Cluster`, `VMSS`, `AVD Pooled Host Pool`, `Azure Databricks`, `Standalone` or the appliance vendor).

A generic VMSS member shows `Manual Review Required` as its execution method, and for both gates in Step 2, until you confirm it's really a stateless pool.

> Patterns **F** (stateful/clustered, including SAP) and **D** are not detected, and for **C** the plugin only detects Databricks. Review your `E` rows for domain controllers, clusters, databases and SAP, and identify other C and D services yourself. To tell them apart: if you can change the size on the existing resource, even with a restart, it's C; if the size is fixed at creation, it's D.

## Step 2: Readiness signals (Assess)

Now check what has to change before the workload can run on the target series:

- **Hyper-V Generation**: the VM's boot generation (Gen 1 or Gen 2), not its size series. v6/v7 require Generation 2 (UEFI), so Gen 1 needs an upgrade or a rebuild from a Gen 2 image. v2 and v3 are often Gen 1, v4 and v5 often Gen 2, so confirm each VM's generation early.
- **Disk Controller Type**: SCSI or NVMe. Moving to NVMe means the OS, drivers and disk-path scripts must support it.
- **Series Generation**: whether the size is already v6/v7.
- **Azure Disk Encryption**: whether ADE is enabled on any attached disk. ADE isn't available on v6/v7 and retires on September 15, 2028. You can't convert an ADE VM in place, and encryption at host is blocked on any VM that has ever had ADE. Linux with an encrypted OS disk needs a new VM and an application-level data migration. On Windows, disable ADE and verify the volumes are decrypted before you copy the disk.
- **Temp Disk Dependency**: check whether the app uses the local temporary disk (scratch files, SQL Server `tempdb`). The plugin can't tell from the resource configuration, so you have to. It reads `Manual Review Required` for B, E and G (and generic VMSS members) and `N/A` for the other A rows and for C. The guide also asks for this check on service-managed compute (C), which the plugin doesn't do, so look at the service's supported sizes. Only `d`-suffixed v6/v7 sizes have a local disk, presented raw as NVMe: pick a `d` size or move that data to a managed disk. Two rules to remember: on Linux the disk is recreated raw at every stop or deallocate, and on Windows you can't resize between sizes with and without a temp disk.
- **SAP/ISV Certification Gate**: `Manual Review Required` for every pattern except A and C, apart from generic VMSS members, which also require review. The plugin can't rule out SAP or an ISV-certified workload, so confirm support for the target VM family yourself.

> `Manual Review Required` means the plugin didn't verify that requirement, not that the VM is incompatible. A firewall appliance with a suggested size still needs vendor confirmation.

SAP (inside F) and appliance (G) certification are hard gates that come before all other planning, not validation items.

## Step 3: Execution method and target SKU (Plan)

For patterns E and G, start with the execution method:

| Situation | Recommendation and what it means |
|-----------|----------------------------------|
| Already on v6/v7 | Nothing to do. |
| Generation 2, older series | **Redeploy**: create a replacement VM from a current, compatible image, then move the application and its data. Review disk compatibility (SCSI-to-NVMe). |
| Generation 1, older series | **Manual Review Required**: rebuild from a Generation 2 image or use a supported Generation 1 to Generation 2 / Trusted Launch upgrade. A size change alone isn't enough, and Gen 1 Dv2/Dv3 VMs can't be upgraded in place, so rebuild those. |
| Pools, AVD pooled hosts or service-managed compute (A, B, C) | Change it through the owning pool or service, not VM by VM. For example, replace an AKS node pool using the AKS process. |

These are starting points, not approval to migrate: **Redeploy** doesn't remove the manual checks from Step 2.

The **target SKU** comes from the same embedded capability catalog behind [`alternative-vm-sku`](/2026/07/29/find-alternative-azure-vm-skus-with-azqr/), and its score measures hardware similarity only (vCPU, memory and features). It's a candidate to validate, not a readiness verdict: it doesn't check CPU architecture, temp-disk needs, disks, NICs, vendor certification, image readiness, region or zone availability, quota or capacity.

The report also flags retirements and price changes, but only for the series in the plugin's embedded dataset (D, Dv2, Dv3, Ev3, A, B, F, G, L and a few HPC/NP series); elsewhere `N/A` means none was encoded.

For costs, the plugin uses the Azure Retail Prices API to estimate the **current SKU's** monthly pay-as-you-go cost in USD (hourly price × 730), then applies price changes that haven't taken effect yet. **Projected Monthly Cost After Increase (USD)** and **Monthly Cost Impact (USD)** describe that increase on the current SKU; they are not the target SKU's price or migration savings. Stopped VMs that are still allocated are billed; deallocated VMs show `N/A (VM deallocated)`. These are retail estimates, not your final bill.

Reserved Instances and Savings Plans are scoped to a VM family, so replan them. Re-apply Azure Hybrid Benefit on the new VMs too, don't assume it carries over.

## From classification to waves

The plugin gives you the classification; the waves are up to you. Microsoft Learn says to group waves by application or service dependency, not to split tightly coupled tiers across old and new families for long, to move non-production first and to keep waves small enough to validate in one window.

That makes it dependency-first: **the pattern decides *how* a workload moves, dependency decides *which* wave it goes in.** A database cluster (F) and its app tiers (E) go in the same wave plan, each with its own procedure.

To apply this, sort and filter the sheet by `Workload Pattern`, `Pool Membership` and `Resource Group`. The sheet doesn't export tags, so use an external inventory (tags or a CMDB) for the production/non-production split and for application grouping.

| Wave | What goes in | Why |
|------|--------------|-----|
| **Hard gates first** | SAP (F) and appliances (G) | Certification of the exact target family comes before any other planning |
| **Wave 0: Pilot** | One representative pattern-E VM, ideally non-production and Gen 2 | Proves the pattern the later waves repeat |
| **v3 milestone** | Dv3, Dsv3, Ev3 and Esv3 VMs, any pattern | Retire on November 15, 2029: early waves, separate milestone |
| **Application waves** | Everything one application depends on, non-production first. Each member follows its pattern: E redeploy or in-place, F add and replace nodes, G parallel appliances | Related tiers move together, so none lingers on the old family |
| **Pool track (A, B, C)** | AKS/VMSS/AVD/Databricks rows | Not per-VM work: replace the pool or cluster with your service's process, in its application's wave |

Inside a wave I still start with the easy VMs (Gen 2, NVMe-ready, no temp disk). The v3 milestone is the exception: many v3 VMs are Gen 1 with a temp disk, and they still go early. Leave time for a final wave and its rollback window before the retirement date.

For clusters, **never modernize all quorum members in one wave**, confirm the target size is available in every zone the cluster spans (not just the region) and plan quota for old and new nodes side by side.

Persistent desktops (E) often arrive hibernated, so resume and deallocate them first. The guide lists no v6 or v7 sizes as hibernation-capable yet.

For **E, F and G**, follow the Microsoft Learn per-VM runbook: pilot, execute, validate, and keep the old VMs (or restore points for in-place upgrades) until the rollback window closes. For **A, B, C and manually identified D**, use the owning service's procedure; for B, assess the golden image first.

### Redeploy or in-place?

Redeploy from a current Gen 2, NVMe- and MANA-ready image is the default: rollback is just keeping the old VM. The in-place upgrade (SCSI-to-NVMe conversion, plus the Gen 1 to Trusted Launch upgrade for eligible sources) is for when you can't practically rebuild. The conservative route uses a source VM without a temp disk or Azure Disk Encryption, with an OS that is already NVMe-ready. Gen 1 sources also need the Trusted Launch prerequisites: no Windows Server 2016 or older, Azure Backup on the Enhanced policy and an unencrypted OS volume. The report signals aren't a full eligibility test.

Rollback depends on the step. The SCSI-to-NVMe conversion reverts by switching back to SCSI and the original size. The Trusted Launch upgrade doesn't revert: you need a full restore from the pre-upgrade backup.

The platform operations are supported, but the community SCSI-to-NVMe script isn't an official Microsoft tool. Test it in non-production, with a backup and a rollback plan, first.

## Scope and limitations

- The plugin covers **Discover, Assess and Plan** only. Validate is on you.
- ARO nodes are recognized by cluster ownership only, so they aren't split into workers and control plane. Hosted-control-plane ARO and self-managed OpenShift aren't covered. Use supported ARO procedures, contact support before control-plane changes and never stop or deallocate cluster VMs directly.
- Failed lookups are logged as warnings, so missing permissions can leave a VM with the default `E` classification. Double-check unexpected `E` rows.
- The temp-disk gate is `Manual Review Required` for B, E and G, and the certification gate for everything except A and C. Generic VMSS members (A) get both. Resolve them before acting on a Redeploy recommendation, starting with SAP and appliance certification.

Hope it helps!

**References:**

* [Modernize to the v6 and v7 VM series](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/lifecycle/sizes-v6-v7-modernization-overview)
* [Modernize to the v6 and v7 VM series with a wave-based runbook](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/lifecycle/sizes-v6-v7-modernization-modernize)
* [Discover modernization pattern by workload type](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/lifecycle/sizes-v6-v7-modernization-discover)
* [Plan your v6 and v7 modernization (wave sequencing)](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/lifecycle/sizes-v6-v7-modernization-plan#wave-sequencing)
* [Assess readiness for the v6 and v7 series](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/lifecycle/sizes-v6-v7-modernization-assess)
* [azqr internal plugins documentation](https://azure.github.io/azqr/docs/plugins/internal-plugins/)
* [Azure Quick Review on GitHub](https://github.com/Azure/azqr)
* [Convert a VM from SCSI to NVMe in place](https://learn.microsoft.com/en-us/azure/virtual-machines/migration/scsi-to-nvme-migration)
