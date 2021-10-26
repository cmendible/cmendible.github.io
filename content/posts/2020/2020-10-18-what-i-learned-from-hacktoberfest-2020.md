---
author: Carlos Mendible
categories:
- kubernetes
date: "2020-10-18T10:00:00Z"
description: 'What I Learned From Hacktoberfest 2020'
images: ["/assets/img/posts/hacktoberfest2020.png"]
published: true
tags: ["hacktoberfest"]
title: 'What I Learned From Hacktoberfest 2020'
---

[Hacktoberfest®](https://hacktoberfest.digitalocean.com/) is an open global event where people all around de globe contribute to open source projects.

The idea behind [Hacktoberfest®](https://hacktoberfest.digitalocean.com/) is great, in my opinion it encourages and motivates contributions specially from those who don't know where to start with OSS, but saddly what we saw this year was many people, let's call them trolls, spamming repos with useless pull requests in order to claim the nice tee. The [Hacktoberfest®](https://hacktoberfest.digitalocean.com/) organization reacted quickly to fix the situation and the rules of the game have been changed: [the event is now offically opt-in only for projects and mantainers](https://hacktoberfest.digitalocean.com/hacktoberfest-update).

Now let me tell you a bit of what I did:

## What I Learned From Hacktoberfest

### Azure Arc enabled Kubernetes:

I learned how to enable container monitoring on [Azure Arc enabled Kubernetes](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/overview?WT.mc_id=AZ-MVP-5002618) and in the process I noticed that in the documentation a powershell command was using `curl` instead of `Invoke-WebRequest` so I sumbited the following PR: [Replacing wget with iwr in the powershell sample](https://github.com/MicrosoftDocs/azure-docs/pull/63625)

### K8Spin

I continued learning about [K8Spin](https://github.com/k8spin/k8spin-operator): a nice OOS project which adds three new hierarchy concepts (Organizations, Tenants, and Spaces) to your kuberneets clusters in order to enable multi-tenancy.

Collaborating with the [K8Spin](https://github.com/k8spin/k8spin-operator) team was great and I fixed two issues:

* A broken link in one of their repos: [Fixing Issue#1: oneinfra link](https://github.com/k8spin/oneinfra-worker-modules/pull/3)
* Added a Helm chart to install the [K8Spin](https://github.com/k8spin/k8spin-operator). I had created some charts in the past but for this one I had to learn about [chart best practices](https://helm.sh/docs/chart_best_practices/) and apply them.

As a bonus also added a PR to add a developer container to their repo in other to enable [Codespaces](https://github.com/features/codespaces) and help any newcomer to start contributing avoiding python and other dependencies setup: [Adding dev container definition](https://github.com/k8spin/k8spin-operator/pull/21)

### Bonus

* I continued my journey learning and collaborating with Dapr and this time I submited a PR to remove [a component from Dapr](https://github.com/dapr/components-contrib/pull/496)
* I hade some SEO issues with my blog and it turns out the the theme I use was not adding [the post title as part of the correspoing tag(https://github.com/avianto/hugo-kiera/pull/41).

Glad to Help!