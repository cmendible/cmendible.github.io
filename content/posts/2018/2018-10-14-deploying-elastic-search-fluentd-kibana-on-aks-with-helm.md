---
author: Carlos Mendible
categories:
- kubernetes
- azure
crosspost_to_medium: true
date: "2018-10-14T14:00:00Z"
description: Deploying Elastic Search, Fluentd, Kibana on AKS with Helm
images: ["/assets/img/posts/kubernetes.png"]
published: true
tags: ["aks", "elasticsearch", "kibana", "fluentd"]
title: Deploying Elastic Search, Fluentd, Kibana on AKS with Helm
---

For my recent talk at [.NET Conf Madrid](http://netconfmad2018.azurewebsites.net/) I managed to install Elastic Search, Fluentd and Kibana (EFK) as a logging solution for the AKS cluster I used in my demos.

The fact is that such deployment was possible thanks to Tim Park and his post [Logging with Fluentd, ElasticSearch, and Kibana (EFK) on Kubernetes on Azure](https://medium.com/@timfpark/efk-logging-on-kubernetes-on-azure-4c54402459c4) where I learned that to effectively deploy EFK on AKS I would have to tweak the resource definitions found in the [Kubernetes](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/fluentd-elasticsearch) project.

I followed Tim's advice and took it to the next level creating my first Helm chart: [efk-aks](https://github.com/cmendible/kubernetes.samples/tree/main/13.efk.helm) so now you can deploy the EFK stack on AKS using Helm.

Feel free to get the Helm chart [here](https://github.com/cmendible/kubernetes.samples/tree/main/13.efk.helm).