---
author: Carlos Mendible
categories:
- azure
date: "2026-08-18"
description: 'Azure Quick Review (azqr) v4.0.0 adds VM SKU recommendations, Service Health analysis, smarter SQL EOL reporting, and quota-aware region selection. Learn what changed and how to prepare your automation before upgrading.'
draft: false
images: ["/assets/img/posts/azqr_readme.png"]
tags: ["azqr", "azure"]
title: 'Azure Quick Review (azqr) v4.0.0: What You Need to Know'
---

[Azure Quick Review (azqr)](https://github.com/Azure/azqr) v4.0.0 is now available. This release expands azqr beyond best-practice assessments with new tools for VM sizing, Service Health analysis, SQL Server lifecycle management, and region selection.

It also includes breaking changes, so if you are upgrading from v3.x, review your scripts and automation before replacing the binary.

> v4.0.0 brings more operational context into the same workflow you already use for Azure governance and compliance reviews.

## TL;DR

The main changes in v4.0.0 are:

- New `alternative-vm-sku` command for ranked VM SKU recommendations
- New Service Health scanner plugin with MCP server integration
- Improved SQL EOL and host-level ESU billing analysis
- Quota and capacity reservation checks in the region selection plugin
- New Service Health prompts and tools in the azqr MCP server
- Three breaking CLI and plugin changes

## Find Alternative VM SKUs

The new `alternative-vm-sku` command uses azqr's embedded VM SKU catalog to find compatible alternatives for a given Azure VM size. Recommendations are ranked using factors such as vCPU count, memory, VM family, generation, GPU count, data disk support, and accelerated networking.

```bash
azqr alternative-vm-sku --sku Standard_D4s_v5 --top 5
```

This is useful when a SKU is unavailable in a target region, a VM family is approaching retirement, or quota constraints require a different size.

For a detailed explanation of the compatibility score and output, read [Find Alternative Azure VM SKUs with azqr](/posts/2026/2026-07-29-azqr-alternative-vm-sku/).

## Add Service Health to Your Review

The new Service Health scanner plugin retrieves Azure Service Health events as part of the azqr review workflow.

Service Health support is also available through the azqr MCP server. AI assistants can use the new prompts and tools to include current health information when analyzing an Azure environment.

## Improve SQL Server Lifecycle Analysis

The SQL EOL plugin now detects Extended Security Updates (ESU) billing at the host level. This matters when a host runs multiple SQL Server instances because ESU charges can depend on the operating system environment, SQL Server version, and highest edition installed.

Arc-enabled machines are now classified using their Cloud Provider metadata, improving the identification of on-premises and multicloud SQL Server hosts.

For details about the report, assumptions, and cost model, read [Identify SQL Server EOL & ESU Costs with azqr](/posts/2026/2026-07-23-azqr-sql-esu-plugin/).

## Check Quota and Capacity Before Choosing a Region

The region selection plugin now includes Azure quota availability and capacity reservations in its analysis. These checks help teams identify deployment constraints before committing to a region.

This is especially useful for workloads that require specific VM families or reserved capacity. A region can support a service and still be unsuitable when the required quota or capacity is unavailable.

## Breaking Changes

Before upgrading from v3.x, update any scripts, pipelines, or agent instructions that use the following commands:

| v3.x | v4.0.0 | Required action |
|------|--------|-----------------|
| `azqr sql-esu` | `azqr sql-eol` | Rename the plugin command |
| `azqr openai-throttling` | `azqr ai-gov` | Rename the plugin command |
| `azqr show` | Removed | Migrate to another reporting workflow |

Use this checklist before rolling out v4.0.0:

- [ ] Search automation for `sql-esu` and replace it with `sql-eol`
- [ ] Search automation for `openai-throttling` and replace it with `ai-gov`
- [ ] Remove dependencies on the `show` command
- [ ] Test scheduled scans and CI/CD pipelines with v4.0.0
- [ ] Update internal documentation and agent instructions

## Install or Upgrade azqr

Install the latest version with the project installation script:

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/azure/azqr/main/scripts/install.sh)"
```

Then verify the installed version:

```bash
azqr version
```

azqr is free and open source. Try the new release, report issues, and contribute improvements through the [Azure Quick Review repository](https://github.com/Azure/azqr).

Hope it helps!

**References:**

* [Azure Quick Review v4.0.0 release notes](https://github.com/Azure/azqr/releases/tag/v.4.0.0)
* [Azure Quick Review documentation](https://azure.github.io/azqr/)
* [Azure Service Health overview](https://learn.microsoft.com/en-us/azure/service-health/overview)
* [Azure subscription and service limits, quotas, and constraints](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits)
* [On-demand capacity reservation overview](https://learn.microsoft.com/en-us/azure/virtual-machines/capacity-reservation-overview)
