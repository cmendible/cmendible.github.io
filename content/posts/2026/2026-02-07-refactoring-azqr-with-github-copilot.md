---
author: Carlos Mendible
categories:
- azure
date: "2026-02-07"
description: 'How I refactored Azure Quick Review (azqr) for maintainability and performance using GitHub Copilot'
draft: false
images: ["/assets/img/posts/azqr_readme.png"]
tags: ["azqr", "github-copilot", "refactoring", "go", "performance", "ai"]
title: 'Refactoring Azure Quick Review with GitHub Copilot'
---

In this post, I'll walk you through the major refactoring of [Azure Quick Review (azqr)](https://github.com/azure/azqr), where I used **GitHub Copilot's plan mode and agent mode** while supervising every change. My role was purely architectural: I defined what needed to change, reviewed every proposal, and guided the AI through the process.

## TL;DR

I refactored [Azure Quick Review (azqr)](https://github.com/azure/azqr) without writing a single line of code by using GitHub Copilot’s plan mode (to design the architecture) and agent mode (to implement it). The refactor eliminated massive technical debt, 72 scanner packages, 72 command files, hundreds of ARM calls and replaced them with a centralized scanner registry, batched Azure Resource Graph queries, a modular pipeline, dynamic command generation, and unified throttling policy.

The result:

* 560 files changed
* 16,506 net lines removed
* ~50% faster scans with costs
* ~90% faster scans without costs
* Cleaner architecture

Copilot acted as the implementer; I acted as the architect and reviewer. The workflow proved that AI can reliably execute impressive refactors when guided by a clear architectural vision.

## The Problem: A Growing Codebase and Not so Quick.

`azqr` scans Azure subscriptions against Azure best practices including the [Well-Architected Framework (WAF)](https://learn.microsoft.com/en-us/azure/well-architected/), the [Cloud Adoption Framework (CAF)](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/) while also downloading useful information from Azure Advisor, Defender, and more. This helps teams identify reliability, security, observability and governance gaps. 

Over time, [Azure Quick Review (azqr)](https://github.com/azure/azqr) codebase had accumulated significant technical debt:

- **72 nearly identical command files** (`cmd/azqr/commands/aks.go`, `apim.go`, `cosmos.go`, ...) each one a hand-crafted Cobra command with the same boilerplate pattern
- **72 individual scanner packages** under `internal/scanners/`, each with its own Go file, rules file, and test file, for a total of **199 files** of per-service scanning logic
- A **705-line monolithic `scanner.go`** that orchestrated everything in a single function with **72 blank imports**
- Each scanner created its own **individual ARM SDK client**, resulting in hundreds of sequential API calls per scan
- No centralized rate limiting or throttling protection strategy

The tool worked, but adding a new Azure service scanner meant touching multiple files, copying boilerplate, and double checking to make sure that nothing broke. Performance and maintainability was degraded with every new service added.

## The Refactoring: Five Architectural Changes

I used **GitHub Copilot's plan mode** to design each refactoring step, then switched to **agent mode** to implement them. Here's what changed across **560 files**:

### 1. From 72 Scanner Packages to a Centralized Registry

**Before:** Every Azure service had its own package with boilerplate initialization, SDK client creation, and rule evaluation:

```go
// internal/scanners/aks/aks.go (OLD: one of 72 packages like this)
package aks

type AKSScanner struct {
    config         *models.ScannerConfig
    clustersClient *armcontainerservice.ManagedClustersClient
}

func (a *AKSScanner) Init(config *models.ScannerConfig) error {
    a.config = config
    a.clustersClient, err = armcontainerservice.NewManagedClustersClient(
        config.SubscriptionID, config.Cred, config.ClientOptions)
    return err
}

func (a *AKSScanner) Scan(scanContext *models.ScanContext) (
    []*models.AzqrServiceResult, error) {
    clusters, err := a.listClusters()
    // ... iterate, evaluate rules, build results
}
```

**After:** A single registry file declares all scanners with zero boilerplate:

```go
// internal/scanners/registry/scanners.go (NEW: replaces 199 files)
func init() {
    models.ScannerList["aks"] = []models.IAzureScanner{
        models.NewBaseScanner("Azure Kubernetes Service",
            "Microsoft.ContainerService/managedClusters"),
    }
}
```

**Impact:** 199 scanner files deleted. One file replaced them all.

### 2. From Individual ARM SDK Calls to Azure Resource Graph

**Before:** Each scanner created its own ARM SDK client and made individual API calls per resource type per subscription. Scanning 50 resource types across 10 subscriptions meant **hundreds of API calls**.

**After:** A single `GraphQueryClient` uses [Azure Resource Graph](https://learn.microsoft.com/en-us/azure/governance/resource-graph/overview) with KQL queries, batching up to **300 subscriptions per query** and processing results with **10 concurrent worker goroutines**:

```go
// internal/graph/graph.go (NEW)
func (q *GraphQueryClient) Query(ctx context.Context, query string,
    subscriptions []*string) *GraphResult {
    // Batch subscriptions in groups of 300
    batchSize := 300
    for i := 0; i < len(subscriptionIDs); i += batchSize {
        // Automatic pagination with skipToken for large result sets
        // ...
    }
}
```

All recommendations are now defined as embedded **KQL files** (e.g., `aks-003.kql`, `sql-004.kql`) that the graph scanner executes in parallel:

```kusto
resources
| where type =~ 'Microsoft.ContainerService/managedClusters'
| where properties.apiServerAccessProfile.enablePrivateCluster != true
| extend recommendationId = 'aks-004'
| extend param1 = 'Public cluster'
| project recommendationId, name, id, tags, param1
```

**Impact:** Dramatic reduction in API calls, from hundreds of individual ARM calls to batched & parallel Resource Graph queries. Scans complete significantly faster.

The migration of every old ARM SDK scanner to KQL queries, including the SLA calculations that required interpreting SKU tiers, redundancy options, and zone configurations for each service, was **entirely AI-driven**. I did not hand-write a single KQL query. Instead, I used GitHub Copilot agent mode to translate the Go-based rule evaluation logic into equivalent KQL, then validated the results using **multiple AI agents and models** to cross-check correctness before merging.

> This was one of the most impressive parts of the refactoring: translating imperative Go code into declarative KQL queries at scale, with AI doing the heavy lifting and helping verify the output.

### 3. From Monolithic Scanner to Pipeline Pattern

**Before:** A single 705-line `scanner.go` file handled everything: authentication, subscription discovery, resource scanning, advisor checks, defender status, cost analysis, and report generation. All in one tangled function:

```go
// internal/scanner.go (OLD: 705 lines, 72 blank imports)
import (
    _ "github.com/Azure/azqr/internal/scanners/aks"
    _ "github.com/Azure/azqr/internal/scanners/apim"
    _ "github.com/Azure/azqr/internal/scanners/cosmos"
    // ... 69 more blank imports
)

func (sc Scanner) Scan(params *ScanParams) string {
    // 705 lines of sequential logic handling everything
}
```

**After:** A composable pipeline with **14 discrete stages**, each in its own file, orchestrated by a builder pattern:

```go
// internal/pipeline/builder.go (NEW)
func (b *ScanPipelineBuilder) BuildDefault() *Pipeline {
    return b.
        WithProfiling().
        WithInitialization().
        WithSubscriptionDiscovery().
        WithResourceDiscovery().
        WithGraphScan().
        WithDiagnosticsScan().
        WithAdvisor().
        WithDefenderStatus().
        WithDefenderRecommendations().
        WithAzurePolicy().
        WithArcSQL().
        WithCost().
        WithPluginExecution().
        WithReportRendering().
        WithProfilingCleanup().
        Build()
}
```

Each stage implements a simple interface and can be independently tested, skipped, or reordered:

```go
type Stage interface {
    Name() string
    Execute(ctx *ScanContext) error
    Skip(ctx *ScanContext) bool
}
```

**Impact:** The scan flow is now modular, testable, and observable. Each stage tracks its own execution time, and the pipeline logs performance metrics per stage.

### 4. From 72 Command Files to Dynamic Command Generation

**Before:** Every Azure service had its own hand-crafted Cobra command file:

```go
// cmd/azqr/commands/aks.go (OLD: one of 72 identical files)
var aksCmd = &cobra.Command{
    Use:   "aks",
    Short: "Scan Azure Kubernetes Service",
    Run: func(cmd *cobra.Command, args []string) {
        scan(cmd, []string{"aks"})
    },
}

func init() {
    scanCmd.AddCommand(aksCmd)
}
```

**After:** A single `scanners.go` file dynamically generates all subcommands from the registry:

```go
// cmd/azqr/commands/scanners.go (NEW: replaces 72 files)
func init() {
    for _, abbr := range abbreviations {
        scanners := models.ScannerList[abbr]
        serviceName := scanners[0].ServiceName()
        cmd := &cobra.Command{
            Use:   abbr,
            Short: fmt.Sprintf("Scan %s", serviceName),
            Run: func(cmd *cobra.Command, args []string) {
                scan(cmd, []string{abbr})
            },
        }
        scanCmd.AddCommand(cmd)
    }
}
```

**Impact:** 72 command files deleted. Adding a new Azure service scanner now requires adding exactly **one entry** in the registry: no new files, no new commands, no new imports.

### 5. Unified Throttling

**Before:** Semi-centralized rate limiting. API calls could trigger Azure throttling errors, especially with many subscriptions.

**After:** A unified `ThrottlingPolicy` applies API-specific rate limits using the token bucket algorithm:

```go
// internal/throttling/policy.go (NEW)
var armLimiter   = rate.NewLimiter(rate.Limit(20), 100)  // ARM: 20 ops/sec
var graphLimiter = rate.NewLimiter(rate.Limit(3), 10)    // Graph: 3 ops/sec
var costLimiter  = rate.NewLimiter(rate.Limit(0.2), 1)   // Cost: 1 per 5sec

func (p *ThrottlingPolicy) Do(req *policy.Request) (*http.Response, error) {
    url := req.Raw().URL.String()
    switch {
    case strings.Contains(url, "Microsoft.ResourceGraph/resources"):
        err = graphLimiter.Wait(req.Raw().Context())
    case strings.Contains(url, "Microsoft.CostManagement/query"):
        err = costLimiter.Wait(req.Raw().Context())
    default:
        err = armLimiter.Wait(req.Raw().Context())
    }
    return req.Next()
}
```

## Performance: Cutting Scan Time

Beyond the architectural changes, I used **GitHub Copilot** to profile and optimize hot paths across the codebase. When scanning **100 subscriptions with over 7,000 resources**, these optimizations cut total scan time in half with costs enabled, and by **90% when costs are disabled** (the new default).

### Excel Report Generation

The Excel renderer was one of the biggest bottlenecks. The old code created **new style objects for every sheet** and applied styles **cell by cell** in nested loops:

```go
// OLD: Creating a new style per sheet, applying per cell
for i := 5; i <= lastRow; i++ {
    for j := 1; j <= columns; j++ {
        cell, _ := excelize.CoordinatesToCellName(j, i)
        if i%2 == 0 {
            f.SetCellStyle(sheet, cell, cell, blue) // one API call per cell!
        }
    }
}
```

The refactored version creates styles **once** via a `StyleCache` and applies them to **entire row ranges**:

```go
// NEW: Cached styles, applied per row range
styles, _ := createSharedStyles(f) // created once, reused everywhere

for i := 5; i <= lastRow; i++ {
    style := styles.White
    if i%2 == 0 {
        style = styles.Blue
    }
    startCell, _ := excelize.CoordinatesToCellName(1, i)
    endCell, _ := excelize.CoordinatesToCellName(columns, i)
    f.SetCellStyle(sheet, startCell, endCell, style) // one call per row
}
```

The `autofit` function was also optimized to **sample only the first 1,000 rows** with early exit at max width, instead of iterating every cell in every column:

```go
// NEW: Sampled autofit with early exit
sampleSize := len(col)
if sampleSize > maxSampleRows {
    sampleSize = maxSampleRows
}
for i := 0; i < sampleSize; i++ {
    cellWidth := len(col[i]) + 3
    if cellWidth > largestWidth {
        largestWidth = cellWidth
    }
    if largestWidth >= 120 { 
        break
    }
}
```

### Nested Loop Elimination

Several report data methods had **O(n×m) nested loops** that were replaced with **O(n+m) map lookups**. For example, the SLA lookup iterated over the entire graph results for every resource:

```go
// OLD: O(resources × graph) nested loop
for _, r := range resources {
    for _, a := range rd.Graph {
        if strings.EqualFold(a.ResourceID, r.ID) {
            if a.Category == models.CategorySLA {
                sla = a.Param1
            }
        }
    }
}
```

Replaced with a pre-built map for O(1) lookups:

```go
// NEW: O(graph + resources) with map lookup
slaMap := make(map[string]string, len(rd.Graph))
for _, a := range rd.Graph {
    if a.Category == models.CategorySLA {
        slaMap[strings.ToLower(a.ResourceID)] = a.Param1
    }
}
for _, r := range resources {
    sla := slaMap[strings.ToLower(r.ID)]
}
```

### Missing Pointer Receivers

Multiple methods on `GraphScanner` were using **value receivers** instead of pointer receivers, causing the entire struct (including embedded file system data) to be **copied on every call**:

```go
// OLD: Value receiver copies the entire GraphScanner struct
func (a GraphScanner) GetRecommendations() map[string]map[string]models.GraphRecommendation { ... }
func (a GraphScanner) Scan(ctx context.Context, cred azcore.TokenCredential) []*models.GraphResult { ... }

// NEW: Pointer receiver no copy
func (a *GraphScanner) GetRecommendations() map[string]map[string]models.GraphRecommendation { ... }
func (a *GraphScanner) Scan(ctx context.Context, cred azcore.TokenCredential) []*models.GraphResult { ... }
```

Six methods were fixed, eliminating unnecessary struct copies in the hottest scanning paths.

### Slice Pre-allocation and Struct-Key Deduplication

Slices were being created with zero capacity, forcing the Go runtime to repeatedly reallocate and copy as they grew. Deduplication used string concatenation for map keys, allocating a new string on every iteration:

```go
// OLD: Zero-capacity slices and string-concat keys
Data: make([]interface{}, 0)             // grows unpredictably
seen := make(map[string]bool)            // wastes memory (bool vs struct{})
key := rec.ResourceID + "|" + rec.RecommendationID  // allocates on every iteration
```

Replaced with pre-allocated slices and struct-based composite keys:

```go
// NEW: Pre-allocated slices and zero-allocation keys
Data: make([]interface{}, 0, 5000)       // pre-allocated capacity

type advisorKey struct {                  // composite key: no allocations
    resourceID       string
    recommendationID string
    category         string
}
seen := make(map[advisorKey]struct{}, len(result.Data))  // struct{} saves memory
```

This pattern was applied across the codebase, especially in the graph scanning and report generation paths, reducing GC pressure and improving performance.

## The GitHub Copilot Workflow

I used GitHub Copilot **plan mode** and [**agent mode**](https://code.visualstudio.com/blogs/2025/02/24/introducing-copilot-agent-mode) throughout this refactoring, and understanding the difference is key to how I delivered all these changes without writing a single line of code.

### Plan mode: Designing the architecture

Plan mode enforces a **planning-first phase**. When you describe a goal, Copilot doesn't immediately start editing files. Instead, it analyzes the codebase, asks clarifying questions, and produces a **structured implementation plan** for your review. Only after you approve the plan does it move to implementation.

For each major refactoring step, I started in plan mode. For example, I would describe the goal: *"Replace the monolithic scanner.go with a composable pipeline pattern using discrete stages"*. Copilot would then:

- Analyze the existing `scanner.go` structure and its dependencies
- Propose the `Stage` interface design and the list of stages
- Outline which files to create, modify, and delete
- Flag potential risks (e.g., breaking the plugin-only scan path)

I reviewed each plan, refined the scope, and only then moved to execution. This separation between **thinking** and **doing** prevented premature code changes and caught misunderstandings early.

### Agent mode: Executing the plan

[Agent mode](https://learn.microsoft.com/en-us/visualstudio/ide/copilot-agent-mode) is where Copilot becomes an **autonomous coding agent**. It doesn't just suggest completions, it iterates through multi-step tasks on its own: finding relevant files, editing code, running terminal commands, reading build output, and self-correcting when something fails. Every action is visible and requires your approval before it's applied.

With the plan approved, I switched to agent mode and asked Copilot to implement it. For the pipeline refactoring, this meant Copilot autonomously:

- Created all new stage files under `internal/pipeline/`
- Extracted logic from the monolithic `scanner.go` into each stage
- Built the `ScanPipelineBuilder` with its fluent API
- Updated all imports and wiring in the command layer
- Ran `make` and `make test` to verify compilation and fixed errors it found

### The iteration loop

The workflow for each of the commits followed a consistent loop:

1. **Plan mode** — Describe the architectural goal, review the generated plan
2. **Agent mode** — Execute the plan, let Copilot create and modify files
3. **Review** — Inspect the generated diffs, request adjustments via follow-up prompts
4. **Verify** — Copilot runs `make` and `make test` to verify compilation and fixed errors it found
5. **Manual testing** — I ran the CLI against test subscriptions to validate behavior and performance improvements
5. **Commit** — Once satisfied, I commit the changes

My role was **architect and reviewer** defining *what* needed to change and *why*, evaluating every proposal, and steering Copilot when it drifted. Copilot's role was **implementer**, handling the tedious mechanics of creating files, moving code, updating imports, and ensuring everything compiles.

> I’m starting to think I should rename my blog. “Code it Yourself” sounds weird when I’m doing absolutely zero coding. Maybe “Supervise it Yourself"? Open to suggestions. 

## By the Numbers

| Metric | Before | After |
|--------|--------|-------|
| **Total files changed** | — | 560 |
| **Lines added** | — | 11,565 |
| **Lines removed** | — | 28,071 |
| **Net reduction** | — | **16,506 lines** |
| **Scanner packages** | 72 (199 files) | 1 registry (570 lines) |
| **Command files** | 72 | 1 (dynamic generation) |
| **Monolithic scanner** | 705 lines, 1 file | 14 stages, 14 files |
| **Blank imports** | 72 | 1 |
| **API approach** | Individual ARM SDK calls | Batched Resource Graph |
| **Rate limiting** | Broken for some use cases | Per-API token bucket |
| **Scan time (with costs)** | Baseline | **~50% reduction** |
| **Scan time (without costs)** | Baseline | **~90% reduction** |

## Conclusion

This refactoring demonstrates that GitHub Copilot isn't just a code completion tool as it can drive **impressive architectural refactoring** when properly supervised. 

The result is a codebase that's **smaller**, **faster**, and **easier to extend**. Adding a new Azure service scanner went from a multi-file, multi-package effort to almost a single line in a registry file.

> The key was clear architectural vision combined with iterative AI-guided execution.

If you're considering refactoring your own codebase, I'd encourage you to try the plan mode + agent mode workflow. Define the architecture, let Copilot do the heavy lifting, and focus your energy on review and guidance.

Hope it helps!

**References:**

* [Azure Quick Review (azqr) on GitHub](https://github.com/azure/azqr)
* [Azure Resource Graph overview](https://learn.microsoft.com/en-us/azure/governance/resource-graph/overview)
* [Azure Resource Manager request limits and throttling](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/request-limits-and-throttling)
* [Guidance for throttled requests in Azure Resource Graph](https://learn.microsoft.com/en-us/azure/governance/resource-graph/concepts/guidance-for-throttled-requests)
* [GitHub Copilot documentation](https://docs.github.com/en/copilot)
