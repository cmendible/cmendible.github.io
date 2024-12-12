---
author: Carlos Mendible
categories:
- azure
date: "2024-12-12"
description: 'Deploy Flex Consumption Azure Function with VNet Integration using Terraform'
images: ["/assets/img/posts/azurefunctions.jpg"]
draft: false
tags: ["flex consumption", "azure functions", "serverless", "terraform"]
title: 'Deploy Flex Consumption Azure Function with VNet Integration using Terraform'
---

The Flex Consumption plan for Azure Functions is a new hosting option that provides more flexibility and cost efficiency for running serverless applications. Unlike the traditional Consumption plan, which charges based on the number of executions and execution time, the Flex Consumption plan allows you to specify the maximum number of instances and memory allocation for your function app. This plan is ideal for scenarios where you need predictable performance and cost, as it enables you to control the scaling behavior of your functions more precisely.

Another important feature of the Flex Consumption plan is the ability to integrate with a virtual network (VNet). This allows you to securely connect your function app to other resources in your Azure environment, such as databases, storage accounts, and APIs. VNet Integration enables you to access these resources without exposing them to the public internet, enhancing the security and performance of your serverless applications.

In this post, I will walk you through the steps to deploy an Azure Function with VNet Integration using Terraform, leveraging the Flex Consumption plan. I will use the [azapi](https://registry.terraform.io/providers/azure/azapi/latest/docs) provider to deploy the Function App, as the `azurerm_linux_function_app` resource [does not yet support the Flex Consumption plan](https://github.com/hashicorp/terraform-provider-azurerm/pull/27531).

## Deploy Flex Consumption Azure Function with VNet Integration using Terraform

### Providers

Create a file called `providers.tf` with the following contents:

``` bash
terraform {
  required_providers {
    azapi = {
      source = "azure/azapi"
    }
    azurerm = {
      source = "hashicorp/azurerm"
    }
  }
}

provider "azapi" {
}

provider "azurerm" {
  features {}
}
```

### Variables

Create a file called `variables.tf` with the following contents:

``` bash
variable "resource_group_name" {
  description = "The name of the resource group"
  type        = string
  default     = "rg-func-flex"
}

variable "location" {
  description = "The location of the resources"
  type        = string
  default     = "northeurope"
}

variable "function_name" {
  description = "The name of the azure function"
  type        = string
  default     = "func-flex"
}

variable "plan_name" {
  description = "The name of the app service plan"
  type        = string
  default     = "plan-func-flex"
}

variable "storage_account_name" {
  description = "The name of the storage account"
  type        = string
  default     = "stfuncflex"
}

variable "log_name" {
  description = "The name of the log analytics workspace"
  type        = string
  default     = "log-func-flex"
}

variable "appi_name" {
  description = "The name of the application insights"
  type        = string
  default     = "appi-func-flex"
}
```

> Note: Flex Consumption Plan is only available in some regions. You can check availability running the following command: `az functionapp list-flexconsumption-locations --output table`

### Virtual Network and Subnet

Create a file called `vnet.tf` with the following contents:

``` bash
# Create VNET for VNET Integration
resource "azurerm_virtual_network" "vnet" {
  name                = "vnet-flex"
  address_space       = ["10.6.0.0/16"]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_servers         = []
}

# Create the Subnet for VNET Integration
resource "azurerm_subnet" "flex" {
  name                 = "flex"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.6.2.0/24"]

  # Delegate the subnet to "Microsoft.App/environments"
  delegation {
    name = "flex-delegation"

    service_delegation {
      name    = "Microsoft.App/environments"
      actions = ["Microsoft.Network/virtualNetworks/subnets/join/action"]
    }
  }
}
```

> Note: subnet must have enough room for your scaling needs and delegation to `Microsoft.App/environments` is required for VNet Integration with Flex Consumption plan. 

### Storage Account

Create a file called `st.tf` with the following contents:

``` bash
# Create the Storage Account.
resource "azurerm_storage_account" "sa" {
  name                            = local.storage_account_name
  location                        = azurerm_resource_group.rg.location
  resource_group_name             = azurerm_resource_group.rg.name
  account_tier                    = "Standard"
  account_replication_type        = "GRS"
  min_tls_version                 = "TLS1_2"
  allow_nested_items_to_be_public = false
  public_network_access_enabled   = true
  network_rules {
    default_action = "Allow"
    bypass = [ "AzureServices" ]
  }
}

# Create the Storage Container
resource "azurerm_storage_container" "container" {
  name                  = local.container_name
  storage_account_id    = azurerm_storage_account.sa.id
  container_access_type = "private"
}
```

> Note: the container is required to store the application code. 

### Application Insights and Log Analytics Workspace

Create a file called `appi.tf` with the following contents:

``` bash
# Create the Log Analytics Workspace
resource "azurerm_log_analytics_workspace" "logs" {
  name                = var.log_name
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
}

# Create the Application Insights resource
resource "azurerm_application_insights" "appi" {
  name                = var.appi_name
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name
  application_type    = "web"
  workspace_id = azurerm_log_analytics_workspace.logs.id
}
```

### Flex Consumption Plan

Create a file called `plan.tf` with the following contents:

``` bash
# Create the Flex Consumption Plan
resource "azurerm_service_plan" "plan" {
  name                = var.plan_name
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  os_type             = "Linux"
  sku_name            = "FC1" # Flex Consumption Plan
  worker_count        = 1
}
```

> Note: the `azurerm_service_plan` resource does not yet support setting `worker_count` to cero. This will cause an update if you you attempt to redeploy the resources.

### Azure Function with VNet Integration

Create a file called `main.tf` with the following contents:

``` bash
resource "random_id" "random" {
  byte_length = 8
}

# Make sure names are unique
locals {
  name_sufix           = substr(lower(random_id.random.hex), 1, 4)
  resource_group_name  = "${var.resource_group_name}-${local.name_sufix}"
  storage_account_name = "${var.storage_account_name}${local.name_sufix}"
  function_name        = "${var.function_name}-${local.name_sufix}"
  container_name       = "flexcode"
}

# Create the Resource Group
resource "azurerm_resource_group" "rg" {
  name     = local.resource_group_name
  location = var.location
}

# Create the Azure Function with VNet Integration using the azapi provider
resource "azapi_resource" "func" {
  type                      = "Microsoft.Web/sites@2023-12-01"
  schema_validation_enabled = false
  location                  = azurerm_resource_group.rg.location
  name                      = local.function_name
  parent_id                 = azurerm_resource_group.rg.id
  body = {
    kind = "functionapp,linux",
    identity = {
      type : "SystemAssigned"
    }
    properties = {
      serverFarmId           = azurerm_service_plan.plan.id,
      httpsOnly              = true
      virtualNetworkSubnetId = azurerm_subnet.flex.id
      functionAppConfig = {
        deployment = {
          storage = {
            type  = "blobcontainer",
            value = "${azurerm_storage_account.sa.primary_blob_endpoint}${local.container_name}",
            name  = local.container_name,
            authentication = {
              type = "systemassignedidentity"
            }
          }
        },
        scaleAndConcurrency = {
          maximumInstanceCount = 40,
          instanceMemoryMB     = 2048
        },
        runtime = {
          name    = "dotnet-isolated",
          version = "8.0"
        }
      },
      siteConfig = {
        appSettings = [
          {
            name  = "AzureWebJobsStorage__accountName",
            value = azurerm_storage_account.sa.name
          },
          {
            name  = "APPLICATIONINSIGHTS_CONNECTION_STRING",
            value = azurerm_application_insights.appi.connection_string
          }
        ]
      }
    }
  }

  response_export_values = [
    "identity.principalId",
  ]
}

# Assign the Storage Blob Data Owner role to the Function App
resource "azurerm_role_assignment" "storage_roleassignment" {
  scope                = azurerm_storage_account.sa.id
  role_definition_name = "Storage Blob Data Owner"
  principal_id         = azapi_resource.func.output.identity.principalId
}
```

> Note: VNET integration is enabled by setting the `virtualNetworkSubnetId` property in the `properties` block. Also the new Azure Functions properties are set in the `functionAppConfig` block to control scaling and memory allocation as well as the runtime version.

## Deploy the resources

Run the following commands to deploy the resources:

``` bash
export ARM_SUBSCRIPTION_ID=<your-subscription-id>
terraform init
terraform apply
```

Hope it helps!

**References:**

* [Create and manage function apps in the Flex Consumption plan](https://learn.microsoft.com/en-us/azure/azure-functions/flex-consumption-how-to?tabs=azure-cli%2Cvs-code-publish&pivots=programming-language-csharp#create-a-flex-consumption-app)
* [Flex Consumption plan - HTTP trigger to Event Hubs using VNET Integration | Azure Functions](https://github.com/Azure-Samples/functions-e2e-http-to-eventhubs)
* [azurerm_linux_function_app - add flex consumption feature](https://github.com/hashicorp/terraform-provider-azurerm/pull/27531)