---
author: Carlos Mendible
categories:
- kubernetes
crosspost_to_medium: true
date: "2018-03-17T17:23:14Z"
description: My kubectl Cheat Sheet
image: /assets/img/posts/kubernetes.png
published: true
tags: ["CheatSheet", "kubectl"]
title: My kubectl Cheat Sheet
---
This is a small **[kubectl](https://kubernetes.io/docs/reference/kubectl/overview/) Cheat Sheet** with the list of commands and settings I use, almost on a daily basis, when working with kubernetes.

## Get version and cluster information

### Get kubectl version

``` bash
kubectl --version
```

### Get cluster information

``` bash
kubectl cluster-info
```

### Check cluster nodes

``` bash
kubectl get nodes
```

### Get running services

``` bash
kubectl get services -w --all-namespaces
```

## Context

### List all available contexts

``` bash
kubectl config get-contexts
```

### Get current context

``` bash
kubectl config current-context
```

### Change the context

``` bash
kubectl config use-context [context name]
```

## Deployment

### Deploy

``` bash
kubectl apply -f [yaml definition file]
```

### Get deployment definition

``` bash
kubectl get deployment [deployment name] -o yaml
```

### Update the image of a deployment

``` bash
kubectl set image deployment/[deployment name] [container name]=[image tag]
```

### Set autoscale for a deployment

``` bash
kubectl autoscale deployment [deployment name] --min=2 --max=5 --cpu-percent=80
```

### Delete a deployment

``` bash
kubectl delete -f [yaml definition file]
```

### Get secret definition

``` bash
kubectl get secret [secret name] -o yaml
```

### Force delete a pod

``` bash
kubectl delete pod [pod name] --grace-period=0 --force
```

## Logs

### Read a pod's log

``` bash
kubectl logs [pod name] -n [namespace name]
```

## Misc

### Install kubernetes dashboard

``` bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

Hope it helps!