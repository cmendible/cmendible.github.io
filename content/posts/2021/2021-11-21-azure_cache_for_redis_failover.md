---
author: Carlos Mendible
categories:
- azure
date: "2021-11-21T10:00:00Z"
description: 'Azure Cache for Redis: Failover Test'
images: ["/assets/img/posts/redis.png"]
draft: false
tags: ["redis", "availabilty zones"]
title: 'Azure Cache for Redis: Failover Test'
---

Azure Cache for Redis supports zone redundancy in its Premium and Enterprise tiers. A zone-redundant cache runs on VMs spread across multiple Availability Zones. It provides higher resilience and availability.

Today I'll show hot to test the failover of a zone-redundant cache.

## Deploy Azure Cache for Redis with availability zones

### Create a main.tf file with the following content:

``` terraform
terraform {
  required_version = "> 0.14"
  required_providers {
    azurerm = {
      version = "= 2.57.0"
    }
    random = {
      version = "= 3.1.0"
    }
  }
}

provider "azurerm" {
  features {}
}

# Location of the services
variable "location" {
  default = "west europe"
}

# Resource Group Name
variable "resource_group" {
  default = "redis-failover"
}

# Name of the Redis cluster
variable "redis_name" {
  default = "redis-failover"
}

resource "random_id" "random" {
  byte_length = 8
}

resource "azurerm_resource_group" "rg" {
  name     = var.resource_group
  location = var.location
}

resource "azurerm_redis_cache" "redis" {
  name                = "${var.redis_name}-${lower(random_id.random.hex)}"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  capacity            = 2
  family              = "P"
  sku_name            = "Premium"
  enable_non_ssl_port = true
  minimum_tls_version = "1.2"

  redis_configuration {
  }

  zones = ["1", "2"]
}

resource "azurerm_log_analytics_workspace" "logs" {
  name                = "redis-logs"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
}

resource "azurerm_monitor_diagnostic_setting" "monitor" {
  name                       = lower("extaudit-${var.redis_name}-diag")
  target_resource_id         = azurerm_redis_cache.redis.id
  log_analytics_workspace_id = azurerm_log_analytics_workspace.logs.id

  metric {
    category = "AllMetrics"

    retention_policy {
      enabled = false
    }
  }

  log {
    category = "ConnectedClientList"
    enabled  = false

    retention_policy {
      days    = 0
      enabled = false
    }
  }

  lifecycle {
    ignore_changes = [metric]
  }
}

output "redis_name" {
  value = azurerm_redis_cache.redis.name
}

output "redis_host_name" {
  value = azurerm_redis_cache.redis.hostname
}

output "redis_primary_access_key" {
  value     = azurerm_redis_cache.redis.primary_access_key
  sensitive = true
}
```

**Note:** the zones are specified: `zones = ["1", "2"]`, making the cache zone-redundant.

### Deploy the Azure Cache for Redis with availability zones:

Run the following command to deploy the Azure Cache for Redis with availability zones:

``` powershell
terraform init
terraform apply -auto-approve
```

## Test the Azure Cache for Redis failover

### Donwload the redis-cli tool:

``` powershell
Invoke-WebRequest -Uri "https://github.com/microsoftarchive/redis/releases/download/win-3.2.100/Redis-x64-3.2.100.zip" -OutFile redis.zip -UseBasicParsing
Expand-Archive -Path .\redis.zip -DestinationPath .\redis-cli
```

### Use the redis-cli to prepare the cache instance with data:

```powershell
$redis_name=$(terraform output redis_name)
$redis_host_name=$(terraform output redis_host_name)
$redis_primary_access_key=$(terraform output redis_primary_access_key)

.\redis-cli\redis-benchmark -h $redis_host_name -a $redis_primary_access_key -t SET -n 10 -d 1024
```

### Check the availability zone hosting the master node:

``` powershell
$redis_name=$(terraform output redis_name)
az redis show -n $redis_name -g redis-failover --query "instances[?isMaster]"
```

you should get an output similar to:

``` powershell
[
  {
    "isMaster": true,
    "isPrimary": true,
    "nonSslPort": 13000,
    "shardId": 0,
    "sslPort": 15000,
    "zone": "1"
  }
]
```

### Use the redis-cli to execute a long running process:

``` powershell  
.\redis-cli\redis-benchmark -h $redis_host_name -a $redis_primary_access_key -t GET -n 1000000 -d 1024 -c 50
```

### Test the Azure Cache for Redis failover (CLI):

From another terminal, run the following command to test the Azure Cache for Redis failover:

``` powershell
$redis_name=$(terraform output redis_name)
az redis force-reboot --reboot-type PrimaryNode -n $redis_name -g redis-failover
```

**Note:** at the time of writing, the previous command fails with an exception:

``` shell
(InternalServerError) Something went wrong.
RequestID=dececf94-7f11-4ffa-9a4b-35694dd3f091
Code: InternalServerError
Message: Something went wrong.
RequestID=dececf94-7f11-4ffa-9a4b-35694dd3f091
```

Please track the following issue for more information: [az redis force-reboot fails with InternalServerError](https://github.com/Azure/azure-cli/issues/20458)

### Test the Azure Cache for Redis failover (Azure Portal):

To reboot the Primary Node, head to the Azure portal and use the Administration/Reboot section of the Redis cluster:

![Primary Node reboot using the Azure Portal](/assets/img/posts/redis-primary-node-reboot.gif)

Once failover start you should see the long running process disconnect. This means your applications must be able to recover from transient errors when working with the cache.

After the failover is complete, check again which availability zone hosts the primary node:

``` powershell
$redis_name=$(terraform output redis_name)
az redis show -n $redis_name -g redis-failover --query "instances[?isMaster]"
```

the output should be similar to:

``` powershell
[
  {
    "isMaster": true,
    "isPrimary": true,
    "nonSslPort": 13001,
    "shardId": 0,
    "sslPort": 15001,
    "zone": "2"
  }
]
```

Hope it helps!!!
