---
author: Carlos Mendible
categories:
- azure
- kubernetes
crosspost_to_medium: false
date: "2021-10-15T10:00:00Z"
description: 'AKS: Container Insights Pod Resources and Limits'
images: ["/assets/img/posts/azure.png"]
draft: false
tags: ["pod", "resources", "limits"]
title: 'AKS: Container Insights Pod Resources and Limits'
---

Today I'll show you how to use [Container Insights](https://docs.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-overview) and Azure Monitor to check your AKS cluster for pods without resources and limits.

You'll need to use the following tables and fields:

* **KubePodInventory**: Table that stores kubernetes cluster's Pod & container information
  * **ClusterName**: ID of the kubernetes cluster from which the event was sourced
  * **Computer**: Computer/node name in the cluster that has this pod/container.
  * **Namespace**: Kubernetes Namespace for the pod/container
  * **ContainerName**:This is in poduid/containername format.
* **Perf**: Performance counters from Windows and Linux agents that provide insight into the performance of hardware components operating systems and applications.
  * **ObjectName**: Name of the performance object.
  * **CounterName**: Name of the performance counter. 
  * **CounterValue**: The value of the counter

And take a close look at the following Objects and Counters:

ObjectName   | Counter                 | Description |
-------------|-------------------------|-------------|
K8SContainer | cpuLimitNanoCores       | Container’s cpu limit in nanocore/nanocpu unit. If container resource limits are not specified, node’s capacity will be rolled-up as container’s limit. 
K8SContainer | cpuRequestNanoCores     | Container’s cpu request in nanocore/nanocpu unit. If container cpu resource requests are not specified, this metric will not be collected.
K8SContainer | memoryLimitBytes        | Container’s memory limit in bytes. If container resource limits are not specified, node’s capacity will be rolled-up as container’s limit. 
K8SContainer | memoryRequestBytes      | Container’s memory request in bytes. If container memory resource requests are not specified, this metric will not be collected.
K8SNode      | cpuAllocatableNanoCores | Amount of cpu that is allocatable by Kubernetes to run pods, expressed in nanocores/nanocpu. 
K8SNode      | cpuCapacityNanoCores    | Total CPU capacity of the node in nanocore/nanocpu unit.
K8SNode      | memoryAllocatableBytes  | Amount of memory in bytes that is allocatable by kubernetes to run pods. 
K8SNode      | memoryCapacityBytes     | Total memory capacity of the node in bytes.

Now let's cut the chace, run the following KQL query on Azure Monitor and check the results:

``` shell
let podCounters = Perf 
    | where ObjectName == 'K8SContainer' and  (CounterName == 'cpuLimitNanoCores' or CounterName == 'cpuRequestNanoCores' or CounterName == 'memoryLimitBytes' or CounterName == 'memoryRequestBytes') 
    | summarize d = make_bag(pack(CounterName, CounterValue)) by InstanceName
    | evaluate bag_unpack(d);
let podResourcesAndLimits = podCounters
    | extend InstanceNameParts = split(InstanceName, "/")
    | extend PodUI = tostring(InstanceNameParts[(array_length(InstanceNameParts)-2)]) 
    | extend PodName = tostring(InstanceNameParts[(array_length(InstanceNameParts)-1)])
    | project PodUI, PodName, cpuLimitNanoCores, cpuRequestNanoCores, memoryLimitBytes, memoryRequestBytes;
let nodeCounters = Perf 
    | where ObjectName == "K8SNode" and  (CounterName == 'cpuAllocatableNanoCores' or CounterName == 'cpuCapacityNanoCores' or CounterName == 'memoryAllocatableBytes' or CounterName == 'memoryCapacityBytes')
    | summarize d = make_bag(pack(CounterName, CounterValue)) by InstanceName
    | evaluate bag_unpack(d);
let nodeCapacity = nodeCounters
    | extend InstanceNameParts = split(InstanceName, "/")
    | extend Computer = tostring(InstanceNameParts[(array_length(InstanceNameParts)-1)])
    | project-away InstanceNameParts, InstanceName;
KubePodInventory
    | distinct ClusterName, Computer, Namespace, ContainerName
    | extend InstanceNameParts = split(ContainerName, "/") 
    | extend PodUI = tostring(InstanceNameParts[(array_length(InstanceNameParts)-2)])
    | extend PodName = tostring(InstanceNameParts[(array_length(InstanceNameParts)-1)])
    | project ClusterName, Computer, Namespace, PodUI, PodName
    | join kind= leftouter (nodeCapacity) on Computer
    | join kind= leftouter (podResourcesAndLimits) on PodUI, PodName
      // Pods without CPU Requests. If container cpu resource requests are not specified, cpuRequestNanoCores metric will not be collected
    | extend CPURequests = isnotnull(cpuRequestNanoCores)
      // Pods without CPU Limits. If container resource limits are not specified, node's capacity will be rolled-up as container's limit
    | extend CPULimits = cpuAllocatableNanoCores != cpuLimitNanoCores 
      // Pods without Memory Requests. If container memory resource requests are not specified, memoryRequestBytes metric will not be collected
    | extend MemoryRequests = isnotnull(memoryRequestBytes) 
      // Pods without Memory Limits. If container resource limits are not specified, node's capacity will be rolled-up as container's limit
    | extend MemoryLimits = memoryAllocatableBytes != memoryLimitBytes 
    | distinct ClusterName, Namespace, PodName, CPURequests, CPULimits, MemoryRequests, MemoryLimits
    | where not(CPURequests) or not(CPULimits) or not(MemoryRequests) or not(MemoryLimits)
    | project ClusterName, Namespace, PodName, CPURequests, CPULimits, MemoryRequests, MemoryLimits
```

Hope it helps!!!

Please find the KQL file [here](https://github.com/cmendible/azure.samples/tree/main/aks_pod_resource_and_limits)

References:

* [Container insights overview](https://docs.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-overview)
* [Azure monitor for containers — metrics & alerts explained](https://medium.com/microsoftazure/azure-monitor-for-containers-metrics-alerts-explained-814e4ed623b)
* [Perf](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/perf)
* [KubePodInventory](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/kubepodinventory)
