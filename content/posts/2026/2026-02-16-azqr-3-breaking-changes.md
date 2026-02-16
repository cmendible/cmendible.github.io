---
author: Carlos Mendible
categories:
- azure
date: "2026-02-16"
description: 'A complete guide to migrating your scripts and automation from Azure Quick Review (azqr) v2.16.1 to v3.0.0'
draft: false
images: ["/assets/img/posts/azqr_readme.png"]
tags: ["azqr", "azure", "cli", "migration"]
title: 'Azure Quick Review (azqr) 3.0: Breaking Changes and Migration Guide'
---

[Azure Quick Review (azqr)](https://github.com/azure/azqr) version 3.0.0 is here with a major architectural refactoring. While the core functionality remains the same, the CLI has been redesigned to provide better flexibility and control over scan stages.

> If you have scripts or automation using azqr, this post will help you migrate to the new version.

## TL;DR

The main changes in v3.0.0:

- Individual boolean flags (`--defender`, `--advisor`, `--costs`, etc.) replaced with unified `--stages` flag
- Cost analysis, Defender recommendations and Arc scans are now **disabled by default**
- New `--stage-param` flag for stage-specific options

## CLI Flag Changes

### Removed Flags

The following boolean flags have been removed and replaced with the unified `--stages` flag:

| Removed Flag | v2.16.1 Usage | v3.0.0 Equivalent |
|--------------|---------------|-------------------|
| `--defender` / `-d` | `azqr scan --defender=false` | `azqr scan --stages -defender` |
| `--advisor` / `-a` | `azqr scan --advisor=false` | `azqr scan --stages -advisor` |
| `--costs` / `-c` | `azqr scan --costs=false` | `azqr scan --stages -cost` |
| `--policy` / `-p` | `azqr scan --policy=true` | `azqr scan --stages policy` |
| `--arc` | `azqr scan --arc=false` | `azqr scan --stages -arc` |
| `--azqr` | `azqr scan --azqr=false` | no longer needed |

### New Unified Stage Control

The `--stages` flag provides granular control over scan stages:

```bash
# Enable specific stages (replaces defaults)
azqr scan --stages cost,policy,arc

# Disable specific stages (keeps other defaults enabled)
azqr scan --stages -diagnostics

# Enable all stages
azqr scan --stages advisor,defender-recommendations,arc,policy,cost
```

**Available stages:**

| Stage | Description | Default in v3.0.0 |
|-------|-------------| -------------|
| `advisor` | Azure Advisor recommendations | enabled |
| `defender` | Microsoft Defender status | enabled |
| `defender-recommendations` | Microsoft Defender detailed recommendations | disabled |
| `arc` | Azure Arc-enabled SQL Server instances | disabled |
| `policy` | Azure Policy compliance states | disabled |
| `cost` | Cost analysis for the last calendar month | disabled |
| `diagnostics` | Diagnostic settings scan (includes azqr recommendations) | enabled |

## Default Behavior Changes

### Enabled by Default (v3.0.0)

- `diagnostics` - Diagnostic settings and azqr recommendations
- `advisor` - Azure Advisor recommendations
- `defender` - Microsoft Defender status

### Disabled by Default (v3.0.0)

The following stages must be explicitly enabled:

- `cost` - Cost analysis (**was enabled by default in v2.16.1**)
- `policy` - Azure Policy compliance
- `arc` - Azure Arc-enabled SQL Server
- `defender-recommendations` - Defender recommendations

**Migration example:**

```bash
# v2.16.1 (cost enabled by default)
azqr scan

# v3.0.0 (cost must be enabled explicitly)
azqr scan --stages cost
```

## Excel Report Changes

The Excel output now organizes sheets based on enabled stages:

**Core Sheets (always generated):**
- Recommendations
- ImpactedResources
- ResourceTypes
- Inventory
- OutOfScope

**Optional Sheets (enabled by default):**
- Advisor
- Defender

**Optional Sheets (disabled by default):**
- DefenderRecommendations
- Azure Policy
- Arc SQL
- Costs

## Migration Examples

### Full Scan with All Features

```bash
# v2.16.1
azqr scan --subscription-id <sub-id> --defender=true --advisor=true --costs=true --policy=true

# v3.0.0
azqr scan --subscription-id <sub-id> --stages advisor,defender,cost,policy
```

### Minimal Scan (No Optional Features)

```bash
# v2.16.1
azqr scan --subscription-id <sub-id> --defender=false --advisor=false --costs=false

# v3.0.0
azqr scan --subscription-id <sub-id> --stages -advisor,-defender,
```

## Migration Checklist

Use this checklist to update your scripts and automation:

- [ ] Replace `--defender=false` with `--stages -defender`
- [ ] Replace `--advisor=false` with `--stages -advisor`
- [ ] Replace `--costs=false` with `--stages -cost` (or remove if you want cost disabled)
- [ ] Replace `--policy=true` with `--stages policy`
- [ ] Replace `--arc=false` with `--stages -arc`
- [ ] Add `--stages cost` if you need cost analysis (now disabled by default)
- [ ] Test your scripts with the new default behavior
- [ ] Update CI/CD pipelines and automation scripts

Hope it helps!

**References:**

* [Azure Quick Review (azqr) on GitHub](https://github.com/azure/azqr)
* [Azure Quick Review Documentation](https://azure.github.io/azqr/)
* [Full Changelog v2.16.1 to v3.0.0](https://github.com/Azure/azqr/compare/v.2.16.1...v.3.0.0)
