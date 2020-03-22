---
author: Carlos Mendible
categories:
- azure
- devops
crosspost_to_medium: true
date: "2018-05-21T19:21:39Z"
description: Accepting Azure Marketplace Terms with Azure CLI
image: /assets/img/posts/azure.png
published: true
tags: ["CLI"]
title: Accepting Azure Marketplace Terms with Azure CLI
---

When you try to deploy a VM from the Marketplace using an **ARM** (json) template you'll get an error like the one below in the case when you've not previously accepted the **Legal terms** for the image:

``` json
[{"Legal terms have not been accepted for this item on this subscription. To accept terms using Powershell..."}]
```

Accepting the **Legal terms** is something you have to do once per subscription for each provider offer you want to use. So how can you accept the terms using the **Azure CLI (version 2.0.26 or higher)**?

First get the **urn** of the image you want to deploy:

``` shell
az vm image list --all --publisher paloaltonetworks --offer vmseries --sku bundle1 --query '[0].urn'
```

Now feed the **urn** (i.e. paloaltonetworks:vmseries1:bundle1:8.1.0) to the following command:

``` shell
az vm image accept-terms --urn paloaltonetworks:vmseries1:bundle1:8.1.0
```

And that's it! Hope it helps!
