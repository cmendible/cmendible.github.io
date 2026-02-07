---
author: Carlos Mendible
categories:
- azure
date: "2026-02-01"
description: 'Understanding Azure Zone Mappings with azqr for High-Performance Applications and Disaster Recovery'
draft: false
images: ["/assets/img/posts/azqr_readme.png"]
tags: ["azqr", "availability-zones", "disaster-recovery", "high-availability", "performance"]
title: 'Understanding Azure Zone Mappings with azqr'
---

Azure availability zones are critical for high availability and disaster recovery. However, **zone numbers (1, 2, 3) are logical abstractions—their physical datacenter mappings vary across every subscription**. Your zone-redundant deployment might actually share infrastructure with your DR environment because different subscriptions map zones differently.

This post explores the [Azure Quick Review](https://github.com/azure/azqr) `zone-mapping` plugin and why zone mappings matter for high availability, disaster recovery, and cross-subscription architectures.

> Without verifying zone mappings, your HA architecture may have hidden single points of failure, and your DR strategy may provide no protection at all.

## Prerequisites

Before you begin, ensure you have:

- An Azure subscription
- Azure CLI installed
- Latest version of [azqr](https://github.com/azure/azqr) installed
- Appropriate permissions to query subscription locations

## The Zone Mapping Problem

Availability zones are physically separate datacenters with independent power, cooling, and networking. However, zones use logical numbering (1, 2, 3), and **the logical-to-physical mapping differs per subscription**.

Example:
- Subscription A: Zone 1 → Physical `eastus-az1`
- Subscription B: Zone 1 → Physical `eastus-az3`

Deploying to "Zone 1" across subscriptions doesn't guarantee the same physical datacenter—creating hidden risks in HA and DR architectures.

> **The Risk**: Using the same logical zones across subscriptions without verifying physical mappings can collocate dependent services in the same datacenter, eliminating fault isolation.

### Key Performance Considerations

1. **Database Replication**: Synchronous replication latency varies by physical zone proximity. Identify which zones have <1ms latency for optimal performance.

2. **Application Affinity**: Place application servers in the same physical zone as database primaries to minimize transaction latency (critical for SAP, ERP systems).

### Disaster Recovery: Avoiding the Invisible Single Point of Failure

The problem: Production in Subscription A (Zone 1) and DR in Subscription B (Zone 1) can both map to the same physical datacenter, which means that your DR strategy provides zero protection.

> Make sure your DR zones are physically separate.

## Using azqr zone-mapping Command

The `azqr zone-mapping` command simplifies the process of discovering your subscription's zone mappings. Here's how to use it:

### Install the Latest azqr

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/azure/azqr/main/scripts/install.sh)"
```

### Run the zone-mapping Command

```bash
# Get zone mappings for your subscription
azqr zone-mapping
```

This command queries the Azure Resource Manager API to retrieve availability zone mappings for all regions in your subscription, presenting the information in an easy-to-read format.

## Quick Wins: Essential Best Practices

1. **Document Your Zone Mappings**: Run `azqr zone-mapping` and save the output for reference

   ```bash
   # Generate zone mappings for your subscriptions
   azqr zone-mapping
   
   # Or export to JSON for programmatic access
   azqr zone-mapping --json --stdout > zone-mappings.json
   ```

2. **Compare Across Subscriptions**: Verify production and DR subscriptions have different physical mappings

   ```bash
   # Generate mapping for production subscription
   azqr zone-mapping --subscription prod-sub-id --json --stdout > prod-zones.txt
   
   # Generate mapping for DR subscription  
   azqr zone-mapping --subscription dr-sub-id --json --stdout > dr-zones.txt
   
   # Review side by side to validate physical separation
   diff prod-zones.txt dr-zones.txt
   ```

3. **Add Pipeline Validation**: Integrate azqr zone-mapping checks into CI/CD

   ```bash
   # Example: Validate zones before deployment
   azqr zone-mapping --subscription prod-sub-id --json --stdout | \
       jq -r '.[] | select(.region=="eastus") | .zones[] | select(.logicalZone=="1") | .physicalZone' > primary.txt
   
   azqr zone-mapping --subscription dr-sub-id --json --stdout | \
       jq -r '.[] | select(.region=="eastus") | .zones[] | select(.logicalZone=="2") | .physicalZone' > dr.txt
   
   if [ "$(cat primary.txt)" == "$(cat dr.txt)" ]; then
       echo "ERROR: Primary and DR zones map to same physical datacenter"
       exit 1
   fi
   ```

## Conclusion

Logical zone numbers hide physical reality. Without verification, your zone-redundant architecture may be collocating critical services, and your DR strategy may offer no real protection.

You can use `azqr zone-mapping` to expose the physical topology, then design accordingly.

Hope it helps!

**References:**

* [What are availability zones?](https://learn.microsoft.com/en-us/azure/reliability/availability-zones-overview)
* [Physical and logical availability zones](https://learn.microsoft.com/en-us/azure/reliability/availability-zones-overview#physical-and-logical-availability-zones)
* [Enable Azure VM disaster recovery between availability zones](https://learn.microsoft.com/en-us/azure/site-recovery/azure-to-azure-how-to-enable-zone-to-zone-disaster-recovery)
* [SAP workload configurations with Azure Availability Zones](https://learn.microsoft.com/en-us/azure/sap/workloads/high-availability-zones)
* [Zonal resources and zone resiliency](https://learn.microsoft.com/en-us/azure/reliability/availability-zones-zonal-resource-resiliency)
* [Networking and connectivity for mission-critical workloads on Azure](https://learn.microsoft.com/en-us/azure/well-architected/mission-critical/mission-critical-networking-connectivity)
* [azqr GitHub repository](https://github.com/azure/azqr)
