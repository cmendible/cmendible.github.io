---
author: Carlos Mendible
categories:
- kubernetes
- azure
crosspost_to_medium: true
date: "2021-07-09T10:00:00Z"
description: 'Plan IP addressing for AKS configured with Azure CNI Networking'
images: ["/assets/img/posts/aks.png"]
draft: false
tags: ["cni", "ip"]
title: 'Plan IP addressing for AKS configured with Azure CNI Networking'
---

When configuring Azure Kubernetes Service with Azure Container Network Interface (CNI), every pod gets an IP address of the subnet you've configured. 

So how do you plan you address space? What factors should you consider?

1. Each node consumes one IP.
1. Each pod consumes one IP.
1. Each internal LoadBalancer Service you anticipate consumes one IP.
1. Azure reserves 5 IP addresses within each subnet.
1. The Max pods per node is 250.
1. The Max pods per nodes lower limit is 10.
1. 30 pods es the minimum per cluster.
1. Max nodes per cluster is 1000.
1. When a cluster is upgraded a new node is added as part of the process which requires a minimum of one additional block of IP addresses to be available. Your node count is then n + 1.
1. When you scale a cluster an additional node is added. Your node count is then n + number-of-additional-scaled-nodes-you-anticipate + 1.

With all that in mind the formula to calculate the number of IPs required for your cluster should look like this:

``` shell
requiredIPs = (nodes + 1 + scale) + ((nodes + 1 + scale) * maxPods) + isvc
```

where:

* nodes: Number of nodes (default 3)
* maxPods: Max pods per node (default 30)
* sacale: Number of expected scale nodes
* isvc: Number of expected internal LoadBalancer services

To help you with this I've created a small console program written in golang: **[aksip](https://github.com/cmendible/aksip)** which performs the necessary validations and calculations for you.

Let's say you want a 50 node cluster with one internal load balancer that also includes provision to scale up an additional 10 nodes:

Just run: 

``` shell
aksip -n 50 -s 10
```

The output will show that you'll need 1892 IP addresses and therefore a /21 subnet or larger:

``` json
{
  "nodes": 50,
  "scale": 10,
  "maxPods": 30,
  "isvc": 1,
  "requiredIPs": 1892,
  "cidr": "/21"
}
```

Hope it helps!!!

References:
* [Configure Azure CNI networking in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni)
* [Configure maximum - new clusters](https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni#configure-maximum---new-clusters)
* [Are there any restrictions on using IP addresses within these subnets](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-faq#are-there-any-restrictions-on-using-ip-addresses-within-these-subnets)