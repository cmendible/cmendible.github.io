---
author: Carlos Mendible
categories:
- azure
date: "2026-02-03"
description: 'Diagnose Azure OpenAI throttling with azqr to identify capacity constraints, optimize deployments, and improve load distribution'
draft: false
tags: ["azqr", "azure-openai", "throttling", "capacity-planning", "apim"]
title: 'Diagnose Azure OpenAI Throttling with azqr'
---

As organizations scale their Azure OpenAI workloads, **throttling (HTTP 429 errors) becomes a critical operational concern**. These errors indicate that your requests exceed the provisioned capacity, leading to degraded user experience, failed completions, and potential revenue loss.

This post introduces the [Azure Quick Review](https://github.com/azure/azqr) `openai-throttling` plugin, which helps you identify throttling patterns, analyze affected deployments, and make data-driven decisions for capacity planning.

> Understanding when, where, and why throttling occurs is the first step toward building resilient AI-powered applications.

## Prerequisites

Before you begin, ensure you have:

- An Azure subscription with Azure OpenAI or AI Services resources
- Azure CLI installed and authenticated
- Latest version of [azqr](https://github.com/azure/azqr) installed
- Reader permissions on the target subscription(s)

## The Throttling Problem

Azure OpenAI enforces rate limits based on your deployment's provisioned throughput (PTU) or tokens-per-minute (TPM) quota. When requests exceed these limits, Azure returns **HTTP 429 (Too Many Requests)** errors.

Common throttling scenarios include:

- **Traffic spikes**: Unexpected surges during peak hours
- **Undersized deployments**: Insufficient capacity for workload demands
- **Model contention**: Multiple applications sharing the same deployment
- **Missing spillover**: No backup deployment to handle overflow

> **The Risk**: Without visibility into throttling patterns, you're flying blind—potentially losing requests and frustrating users without knowing the root cause.

## Using the azqr openai-throttling Command

The `azqr openai-throttling` command scans your Azure OpenAI and Cognitive Services accounts, querying Azure Monitor metrics to detect 429 errors over the past 7 days.

### Install the Latest azqr

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/azure/azqr/main/scripts/install.sh)"
```

### Run the openai-throttling Command

```bash
# Standalone (scans all subscriptions)
azqr openai-throttling

# Or for a specific subscription
azqr openai-throttling -s <subscription-id>
```

The plugin queries the `AzureOpenAIRequests` metric with status code dimensions, providing hourly granularity for the past week.

## Analyzing the Results

The output includes detailed information for each hour, deployment, and model:

| Column | Description |
|--------|-------------|
| Subscription | The Azure subscription name |
| Resource Group | Resource group containing the account |
| Account Name | Azure OpenAI or AI Services account |
| Kind | Account type (OpenAI, AIServices) |
| SKU | Pricing tier |
| Deployment Name | The model deployment experiencing requests |
| Model Name | The underlying model (gpt-4, gpt-35-turbo, etc.) |
| Spillover Enabled | Whether spillover is configured |
| Spillover Deployment | Target deployment for overflow traffic |
| Hour | The hour when requests occurred |
| Status Code | HTTP status code (200, 429, etc.) |
| Request Count | Number of requests with that status |

### Key Areas to Focus On

When analyzing results, pay attention to:

**1. Instances Experiencing Throttling**

Filter for rows where `Status Code = 429`. These are your problem areas. Look for:
- Which accounts have the highest 429 counts
- Whether throttling is isolated to specific deployments or widespread

**2. Deployments and Models Affected**

Identify which model deployments are hitting limits:
- High-demand models like `gpt-4` may need more capacity
- Shared deployments serving multiple applications are throttling candidates

**3. Time Patterns of Throttling (Peak Hours)**

Look for temporal patterns:
- Business hours throttling suggests production workload issues
- Batch processing windows may create predictable spikes
- Overnight throttling could indicate scheduled jobs or global users

**4. Spillover Configuration Status**

Check the `Spillover Enabled` column:
- Deployments showing `No` with high 429 counts are candidates for spillover configuration
- Verify spillover deployments have sufficient capacity themselves

## Recommendations

Based on your analysis, consider the following strategies:

### Capacity Planning and Scaling

```bash
# Export data for capacity analysis
azqr openai-throttling --json --stdout > throttling-data.json

# Analyze peak hours and calculate required capacity
cat throttling-data.json | jq '.externalPlugins."openai-throttling".data | [.[] | select(.statusCode == "429")] | group_by(.deploymentName) | map({deployment: .[0].deploymentName, total_requests: map(.requestCount | tonumber) | add})'
```

**Actions:**
- Increase PTU allocation for consistently throttled deployments
- Consider provisioned throughput for predictable workloads
- Request quota increases for TPM-limited deployments

### Load Distribution with Azure API Management

Azure API Management (APIM) can intelligently distribute load across multiple Azure OpenAI instances:

```xml
<!-- APIM policy for round-robin load balancing -->
<policies>
    <inbound>
        <set-backend-service backend-id="openai-backend-pool" />
    </inbound>
</policies>
```

**Benefits:**
- Distribute requests across multiple deployments or regions
- Implement retry logic with automatic failover
- Add caching to reduce redundant requests
- Monitor and rate-limit by application or user

### Spillover Configuration Optimization

For deployments without spillover, configure a backup:

```bash
# Using Azure CLI to update deployment with spillover
az cognitiveservices account deployment create \
  --name <account-name> \
  --resource-group <resource-group> \
  --deployment-name <deployment-name> \
  --model-format OpenAI \
  --model-name gpt-4 \
  --model-version "0613" \
  --sku-capacity 10 \
  --sku-name Standard \
  --spillover-deployment-name <backup-deployment-name>
```

**Best practices:**
- Ensure spillover deployments have adequate capacity
- Consider using different regions for spillover (resilience)
- Monitor spillover usage to detect capacity planning issues

## Integrating with Full Scans

You can combine throttling analysis with a comprehensive Azure review:

```bash
# Run full scan with throttling plugin
azqr scan --plugin openai-throttling --output-name openai-analysis

# View results in interactive dashboard
azqr show -f openai-analysis.xlsx --open
```

## Conclusion

Throttling is inevitable at scale, but it doesn't have to be unpredictable. The `azqr openai-throttling` plugin gives you visibility into your Azure OpenAI capacity constraints, enabling proactive capacity planning instead of reactive firefighting.

Use the data to right-size your deployments, implement intelligent load distribution with APIM, and configure spillover for resilience.

Hope it helps!

**References:**

* [Azure OpenAI Service quotas and limits](https://learn.microsoft.com/en-us/azure/ai-services/openai/quotas-limits)
* [Manage Azure OpenAI Service quota](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/quota)
* [Azure OpenAI Service provisioned throughput](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/provisioned-throughput)
* [Load balance Azure OpenAI with APIM](https://learn.microsoft.com/en-us/samples/azure-samples/openai-apim-lb/openai-apim-lb/)
* [Azure Quick Review Repository](https://github.com/azure/azqr)
