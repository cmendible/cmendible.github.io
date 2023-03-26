---
author: Carlos Mendible
categories:
- azure
- kubernetes
- devops
date: "2019-07-28T00:00:00Z"
description: 'Kured: Restart your Azure Kubernetes Service Nodes'
images: ["/assets/img/posts/aks.png"]
published: true
tags: ["aks", "kured"]
title: 'Kured: Restart your Azure Kubernetes Service Nodes'
---

Two weeks ago I got an email message from Microsoft Azure explaining that Azure Kubernetes Services had been patched but that I had to restart my nodes (reboot the clusters) to complete the operation.

![The Microsoft Azure email explaining that I had to reboot my clusters](https://carlos.mendible.com/assets/img/posts/aks_update_kured.png)

The first thing you need to know is that, when things like this happens, the Azure platform creates a file called `/var/run/reboot-required` in each of the nodes of your cluster.

The second thing is that a Kubernetes Reboot Daemon named [Kured](https://github.com/weaveworks/kured) exists and if installed in your cluster will run on each pod watching for the existence of the `/var/run/reboot-required` file. [Kured](https://github.com/weaveworks/kured) then takes care of the reboots for you so only one node is restarted at a time.

So how do you install it and check what's going on with your nodes? Let's see:

## Check the nodes
---

Damn I was running the affected kernel version: 4.15.0-1049-azure.

``` shell
kubectl get nodes -o wide
```

``` shell
NAME                       STATUS    ROLES     AGE       VERSION   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
aks-agentpool-14502547-0   Ready     agent     3d        v1.12.8   <none>        Ubuntu 16.04.6 LTS   4.15.0-1049-azure   docker://3.0.4
aks-agentpool-14502547-1   Ready     agent     3d        v1.12.8   <none>        Ubuntu 16.04.6 LTS   4.15.0-1049-azure   docker://3.0.4
aks-agentpool-14502547-2   Ready     agent     3d        v1.12.8   <none>        Ubuntu 16.04.6 LTS   4.15.0-1049-azure   docker://3.0.4
```

## Install [Kured](https://github.com/weaveworks/kured)
---

``` shell
kubectl apply -f https://github.com/weaveworks/kured/releases/download/1.2.0/kured-1.2.0-dockerhub.yaml
```

## Check the nodes again
---

``` shell
kubectl get nodes -o wide
```

You can see that the first two nodes already restarted and the second has scheduling disabled cause it was still in the process.

``` shell
aks-agentpool-14502547-0   Ready                      agent     3d        v1.12.8   <none>        Ubuntu 16.04.6 LTS   4.15.0-1049-azure   docker://3.0.4
aks-agentpool-14502547-1   Ready                      agent     3d        v1.12.8   <none>        Ubuntu 16.04.6 LTS   4.15.0-1050-azure   docker://3.0.4
aks-agentpool-14502547-2   Ready,SchedulingDisabled   agent     3d        v1.12.8   <none>        Ubuntu 16.04.6 LTS   4.15.0-1050-azure   docker://3.0.4
```

To learn more about [Kured](https://github.com/weaveworks/kured) and the features it offers, please check [here](https://github.com/weaveworks/kured).

Hope it helps!