---
author: Carlos Mendible
categories:
- azure
date: "2017-02-26T19:39:11Z"
description: Docker Machine with an existing Azure VNET
images: ["/wp-content/uploads/2017/02/dm.png"]
tags: ["docker"]
title: Docker Machine with an existing Azure VNET
---
Last week I had to provision a Docker host and I tried out the **<a href="https://docs.docker.com/machine/" target="_blank">docker-machine</a>** command. The resulting host would have to use an existing Azure subnet from another resource group and I also needed to be able to reach the machine using it's private IP.

After reading the docs and playing for some minutes I came up with the correct command to use **Docker Machine with an existing Azure VNET**:

``` powershell
    docker-machine create --driver azure --azure-subscription-id [subscriptionid]  `
       --azure-resource-group [Resource group] --azure-location [Azure location]  `
       --azure-vnet [Resource group]:[Azure VNET] --azure-subnet [Azure Subnet Name]  `
       --azure-subnet-prefix [Azure subnet prefix (CIDR notation)] --azure-use-private-ip [machine name]
```

Hope it helps!