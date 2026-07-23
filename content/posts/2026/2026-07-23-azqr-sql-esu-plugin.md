---
author: Carlos Mendible
categories:
- azure
date: "2026-07-23"
description: 'Use the azqr sql-eol plugin to identify SQL Server instances approaching end-of-life, estimate ESU costs, and get data-driven migration recommendations to Azure SQL Managed Instance.'
draft: false
images: ["/assets/img/posts/azqr_readme.png"]
tags: ["azqr", "sql-server", "esu", "migration", "cost-optimization"]
title: 'Identify SQL Server EOL & ESU Costs with azqr'
---

Running SQL Server past its end-of-life date exposes your organization to **unpatched security vulnerabilities and growing Extended Security Update (ESU) costs**. Understanding exactly which instances are affected — and what it could cost you — is the first step toward a sound modernization strategy.

This post introduces the [Azure Quick Review](https://github.com/azure/azqr) `sql-eol` plugin (formerly `sql-esu`), which scans both Arc-enabled SQL Servers and Azure SQL VMs, estimates your possible ESU spend, and recommends migration paths to Azure SQL Managed Instance (SQL MI).

> Knowing what you could be paying for ESU — and what you'd save by migrating — turns an abstract compliance concern into a concrete financial decision.

## Prerequisites

Before you begin, ensure you have:

- An Azure subscription with SQL Server resources (Arc-enabled or SQL IaaS VMs)
- Azure CLI installed and authenticated
- Latest version of [azqr](https://github.com/azure/azqr) installed
- Reader permissions on the target subscription(s)

## The SQL Server EOL Problem

Microsoft's mainstream and extended support lifecycle for SQL Server versions eventually ends. After end of support, you have three options:

1. **Do nothing** — run unsupported, unpatched SQL Server (serious security risk)
2. **Purchase ESU** — pay for up to 3 years of critical security patches at escalating cost
3. **Migrate** — move to a modern, fully managed service like Azure SQL Managed Instance

So how do you know which SQL Server instances are approaching end-of-life, and what the ESU costs will be? The `azqr sql-eol` plugin automates this discovery and analysis.

## Using the azqr sql-eol Command

The `azqr sql-eol` command scans your subscriptions using Azure Resource Graph, covering:

- **Arc-enabled SQL Servers** (`microsoft.azurearcdata/sqlserverinstances`) — on-premises or multi-cloud SQL managed by Azure Arc
- **Azure SQL VMs** (`microsoft.sqlvirtualmachine/sqlvirtualmachines`) — SQL Server running on Azure IaaS VMs

### Install the Latest azqr

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/azure/azqr/main/scripts/install.sh)"
```

### Run the sql-eol Command

```bash
# Scan all subscriptions
azqr sql-eol

# Scan a specific subscription
azqr sql-eol -s <subscription-id>
```

Results appear in the **"SQL EOL" sheet** of the generated Excel report.

## Analyzing the Results

The plugin produces a detailed table with 32 columns. Here are the most important ones:

| Column | Description |
|--------|-------------|
| Subscription | Azure subscription name |
| Resource Group | Resource group containing the SQL resource |
| Name | SQL Server instance name |
| Location | Azure region or on-premises location |
| Arc Server Name | Name of the underlying Arc machine (Arc-enabled instances only) |
| Cloud Type | `Arc-enabled (On-Prem)`, `Arc-enabled (AWS)`, `Arc-enabled (GCP)`, `Arc-enabled (<provider>)`, or `Azure VM (SQL IaaS)` |
| Service Type | SQL Server service type (e.g., `Engine`, `SSIS`, `SSAS`, `SSRS`) |
| SQL Version | Detected SQL Server version (e.g., `SQL Server 2014`) |
| Edition | `Enterprise`, `Standard`, `Web`, `Developer`, `Express`, `Free` |
| EOL Status | `Supported`, `Upcoming ESU`, `ESU Active`, or `Expired` |
| ESU Applicable | Whether ESU applies to this instance |
| ESU Enabled | Whether ESU is already subscribed/enabled on the host |
| ESU Start Date | When the ESU window began |
| ESU End Date | When ESU support ends (no more patches after this) |
| vCores / Billable Cores | Core count and Microsoft-minimum billable cores (min 4) |
| SQL License Type | `PAYG` (hourly), `AHUB` (bring-your-own SA license), or `DR` |
| ESU Monthly Cost/Core | Per-core ESU rate used in calculations |
| SQL License Cost/Core/Month | SQL license list rate per core |
| SQL License Monthly Cost | Total SQL license cost (PAYG only; AHUB = $0) |
| VM Cost/Core/Month | Blended VM compute rate per core |
| Est VM Compute Monthly Cost | Estimated VM compute spend ($0 for Arc/on-prem) |
| Est ESU Monthly Cost | Estimated ESU spend ($0 if Supported or Expired) |
| ESU Cost Basis | How ESU billing was determined (per-host or per-instance) |
| Patch Ops Monthly Cost | Fixed monthly operational overhead for patching |
| Current Monthly Cost | Sum of: VM compute + SQL license (PAYG) + ESU + patch ops |
| Consolidation Ratio | Conservative 2:1 source-to-target ratio used in MI cost allocation |
| Migration Target Tier | Recommended SQL MI tier (`General Purpose` or `General Purpose (Database assessment required. Server had an Enterprise license)`) |
| Migration Recommendation | Suggested action for this instance |
| Est SQL MI Monthly Cost | Projected SQL MI cost after migration (compute + license; storage excluded by default) |
| Est SQL MI Monthly Saving | Monthly savings vs. current cost (negative = cost increase) |
| SQL MI Migration Verdict | `Cost Savings`, `Break Even`, or `Cost Increase (justified by PaaS/security benefits)` |

### Key Areas to Focus On

**1. Instances with `EOL Status = ESU Active` or `Expired`**

These are your highest-priority instances. `ESU Active` means you're in the paid ESU window — every month of delay adds to your ESU spend. `Expired` means you're beyond all support, with no security patches at all.

**2. ESU Applicable and ESU Enabled**

`ESU Applicable` tells you whether an instance is eligible for ESU billing. `ESU Enabled` shows whether ESU is already subscribed on the host. An instance that is applicable but not enabled may be running unpatched — a serious security risk.

**3. Host-Level ESU Billing**

Per Microsoft's billing model, ESU is metered **once per OSE (Operating System Environment), per SQL Server version, at the highest edition present**. Multiple same-version instances or components (Engine, SSIS, SSAS, SSRS) on one host are covered by a single charge. The plugin reflects this model via the `ESU Cost Basis` column.

**4. SQL MI Migration Verdict**

The plugin provides a verdict for each instance:

- **Cost Savings** — migrating to SQL MI saves money; act now
- **Break Even** — migration cost roughly equals current spend
- **Cost Increase (justified by PaaS/security benefits)** — SQL MI costs more, but PaaS, security, and operational benefits may still justify the move

> Free editions (`Developer`, `Express`) are excluded from the ESU financial analysis — they do not incur ESU costs.

**5. Est SQL MI Monthly Saving**

Sort by this column descending to identify the highest-value migration targets. Instances with large savings should be at the top of your migration backlog. Note that a **negative value** means SQL MI costs more — common for large AHUB instances not yet in ESU.

**6. Consolidation Ratio**

The plugin applies a conservative **2:1 consolidation ratio** — two SQL Server instances per one SQL MI — reflecting typical SQL Server utilization of only 10–30%. Your actual savings may be even higher.

**7. Cloud Type**

Arc-enabled (on-premises) Enterprise instances with Software Assurance may qualify for the **Unlimited Virtualization Benefit (UVB)**, where 1 on-premises core with SA covers up to 4 SQL MI General Purpose vCores. The plugin accounts for this automatically. The `Cloud Type` column also distinguishes multi-cloud Arc sources: `Arc-enabled (AWS)`, `Arc-enabled (GCP)`, and `Arc-enabled (<provider>)` for any other registered cloud provider.

## Assumptions

The plugin's cost estimates are based on a set of explicit assumptions. Understanding them helps you interpret the numbers correctly — **treat them as indicative, not exact**.

| Assumption | Value / Rule |
|---|---|
| **Pricing basis** | USD public list prices — EA, CSP, or negotiated discounts are **not** applied |
| **ESU cost model** | ESU annual price follows a Year 1 = 75%, Year 2 = 100%, Year 3 = 125% schedule of the annual license cost; the plugin uses the blended 3-year average (~100%) so estimates are stable regardless of which ESU year you are in |
| **ESU rates** | Standard/Web: $139/core/month · Enterprise: $540.50/core/month · Developer/Express/Free: $0 |
| **Minimum billable cores** | 4 cores per instance (Microsoft minimum); Standard edition capped at 24 cores |
| **ESU billing unit** | Once per OSE, per SQL Server version, at the highest edition on that host. All service types (Engine, SSIS, SSAS, SSRS, PBIRS) roll into a single per-host+version meter |
| **VM compute cost** | Blended PAYG by family: M-series $140, E-series $46, L-series $57, F-series $31, D/B-series $36 (per vCore/month). Windows multiplier ×1.8, West Europe ×1.13. Arc/on-prem = $0 |
| **SQL license (PAYG)** | Enterprise: $274 · Standard: $73 · Web: $6 (per vCore/month). AHUB instances carry no hourly charge — SA is a sunk cost |
| **AHUB default for VMs** | If the SQL VM license type is not explicitly set to `PAYG`, the plugin defaults to `AHUB` (most enterprise customers have SA) |
| **Patch ops** | $160/month per instance (2 hrs × $80/hr operational overhead) |
| **SQL MI target tier** | All non-UVB cases → **General Purpose** (conservative estimate). Arc Enterprise AHUB (on-prem) → General Purpose via UVB. Enterprise editions without UVB → General Purpose with a note that a database assessment is required before committing to BC |
| **SQL MI target rates** | GP PAYG: $123/vCore/month · GP AHUB: $49/vCore/month |
| **SQL MI storage** | **Not included by default** (`_estimatedStorageGB = 0`). Set `_estimatedStorageGB` in the KQL to your environment's expected DB size to add storage to the MI cost estimate. GP rate: $0.115/GB/month (e.g. 512 GB → $58.88/month, 1,024 GB → $117.76/month). Free editions: $0 |
| **UVB (Arc Enterprise + AHUB only)** | 1 on-premises core with SA → up to 4 SQL MI GP vCores; sized at max(4, vCores ÷ 4). Azure VMs are excluded — UVB is on-premises only |
| **Consolidation ratio** | Conservative **2:1** (2 SQL Server instances per 1 SQL MI); actual savings may be higher given typical SQL Server utilization of only 10–30% |
| **Savings formula** | `Current (VM compute + SQL license if PAYG + ESU + patch ops) − Est SQL MI cost` |

> Actual Microsoft ESU pricing depends on your agreement type, SQL Server version, and whether you pay annually or monthly. Always verify current rates with Microsoft or your licensing partner before making budget decisions.

Hope it helps!

**References:**

* [SQL Server end of support options](https://learn.microsoft.com/en-us/sql/sql-server/end-of-support/sql-server-end-of-support-overview)
* [Extended Security Updates for SQL Server](https://learn.microsoft.com/en-us/sql/sql-server/end-of-support/sql-server-extended-security-updates)
* [Azure SQL Managed Instance overview](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/sql-managed-instance-paas-overview)
* [Azure Hybrid Benefit for SQL Server](https://learn.microsoft.com/en-us/azure/azure-sql/azure-hybrid-benefit)
* [Azure Database Migration Service](https://learn.microsoft.com/en-us/azure/dms/dms-overview)
* [Azure Arc-enabled SQL Server](https://learn.microsoft.com/en-us/sql/sql-server/azure-arc/overview)
* [Azure Quick Review GitHub repository](https://github.com/azure/azqr)
