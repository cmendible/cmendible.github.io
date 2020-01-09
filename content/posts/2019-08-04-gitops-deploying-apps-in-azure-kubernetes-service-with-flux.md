---
author: Carlos Mendible
categories:
- azure
- kubernetes
- devops
crosspost_to_medium: true
date: "2019-08-04T00:00:00Z"
description: 'GitOps: Deploying apps in Azure Kubernetes Service (AKS) with Flux'
image: /assets/img/posts/aks.png
published: true
tags: ["git", "gitops", "aks"]
title: 'GitOps: Deploying apps in Azure Kubernetes Service (AKS) with Flux'
url: /2019/08/04/gitops-deploying-apps-in-azure-kubernetes-service-with-flux/
---

Recently I learned about GitOps which is a way to manage your Kubernetes clusters and the applications you run on top using Git. The idea is that you can declaratively describe the desired state of your systems in Git and roll out changes as soon as merges occur.

You can immediately see the main benefits of such an approach: Your Git repositories become the single source of truth for both your infrastructure and application code, allowing the teams to increase productivity and stability (you get the Git log to audit changes).

To implement GitOps you can use and configure [Flux](https://docs.fluxcd.io/en/latest/) following some simple steps:

## 1. Download the helm template

``` powershell
helm fetch `
  --repo https://fluxcd.github.io/flux `
  --untar `
  --untardir .\.charts `
  --version 0.10.2 `
  flux
```

## 2. Bake the template with your repo (I don't use Tiller)

``` powershell
helm template flux `
  --set git.url="git@github.com:cmendible/kubernetes.samples" `
  --set git.path="19.flux" `
  --set git.pollInterval="5s" `
  --namespace flux `
  --output-dir .\.baked .\.charts\fluxcd
```

As you can see I'm configuring [Flux](https://docs.fluxcd.io/en/latest/) to use my k8s sample repo and the [19.flux](https://github.com/cmendible/kubernetes.samples/tree/master/19.flux) folder, which contains a simple deployment [file](https://github.com/cmendible/kubernetes.samples/blob/master/19.flux/dni-function.yaml), but of course you can have more resource definitions.

## 3. Deploy the configuration to your cluster

``` powershell
kubectl apply -f .\.baked\flux\templates\
```

## 4. Get the fluxctl CLI

Download the [fluxctl CLI](https://github.com/fluxcd/flux/releases/tag/1.13.2)

## 5. Use fluxctl to get the public key

``` powershell
fluxctl identity --k8s-fwd-ns flux
```

This key is needed to sync your cluster state with the Git repository (GitHub): Copy the key you obtained and use it to create a **deploy key** with write access on your GitHub repository (Settings > Deploy keys > Add deploy key > check Allow write access > paste the Flux public key > click Add key)

You are all set. If everything runs smooth you'll find a new deployment in your cluster with the **dni-function** name.

To learn more about GitOps check: [Weaveworks](https://www.weave.works/technologies/gitops/)

Click here to learn more about [Flux](https://docs.fluxcd.io/en/latest/)

Hope it helps!