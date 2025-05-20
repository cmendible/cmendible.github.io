---
author: Carlos Mendible
categories:
- azure
date: "2025-05-20"
description: 'Deploy Azure AI Foundry and Bing grounding with Terraform'
draft: false
tags: ["aifoundry", "generativeai", "ai"]
title: 'Deploy Azure Foundry and Bing grounding with Terraform'
---

In this post, I'll show you how to deploy Azure AI Foundry connected to Bing Grounding using Terraform. This setup enables you to leverage Azure's powerful AI capabilities and enrich them with Bing's search data, all managed as code for repeatability and automation.

## Prerequisites

Before you begin, make sure you have:

- An active Azure subscription
- [Terraform](https://www.terraform.io/downloads.html) (>= 1.4.6) installed
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) installed and authenticated (`az login`)
- Sufficient permissions to create resources in your Azure subscription

## Overview

We'll use Terraform to provision:

- A resource group
- Azure Key Vault
- Azure Storage Account
- Azure AI Services (for OpenAI models)
- Azure AI Foundry and a Foundry Project
- Bing Account resource (via azapi)
- Connections between Foundry and Bing/OpenAI (via azapi)
- Role assignments for access

## Step 1: Prepare Your Terraform Files

Copy the following code into separate files in your working directory:

- `variables.tf` – for input variables
- `main.tf` – for the main Terraform configuration
- `outputs.tf` – for the output values

### variables.tf:

```bash
# The Resource Group name
variable "resource_group_name" {
  default = "rg-aifoundry-azapi"
}

# The Azure region to deploy resources
variable "location" {
  default = "eastus2"
}

# The Key Vault name
variable "key_vault_name" {
  default = "kv-aifoundry-azapi"
}

# The Storage Account name
variable "storage_account_name" {
  default = "stfoundryazapi"
}

# The AI Services name
variable "ai_services_name" {
  default = "ai-test"
}

# The AI Foundry Hub name
variable "ai_foundry_name" {
  default = "foundry-test"
}

# The AI Foundry Project name
variable "ai_foundry_project_name" {
  default = "aifoundry-project"
}

# The Bing Account name
variable "bing_account_name" {
  default = "bing-ai-foundry"
}
```

### main.tf:

```bash
terraform {
  required_version = ">= 1.4.6"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~>4.0"
    }
    azapi = {
      source  = "Azure/azapi"
      version = "~>1.0"
    }
  }
}

provider "azurerm" {
  features {
    key_vault {
      purge_soft_delete_on_destroy = true
    }
    cognitive_account {
      purge_soft_delete_on_destroy = true
    }
    api_management {
      purge_soft_delete_on_destroy = true
    }
    resource_group {
      prevent_deletion_if_contains_resources = false
    }
  }
}

provider "azuread" {}

data "azurerm_subscription" "current" {}

data "azurerm_client_config" "current" {}

# Create an Azure Resource Group
resource "azurerm_resource_group" "rg" {
  name     = var.resource_group_name
  location = var.location
}

# Create an Azure Key Vault resource
resource "azurerm_key_vault" "kv" {
  name                = var.key_vault_name                          # Name of the Key Vault
  location            = azurerm_resource_group.rg.location          # Location from the resource group
  resource_group_name = azurerm_resource_group.rg.name              # Resource group name
  tenant_id           = data.azurerm_subscription.current.tenant_id # Azure tenant ID

  sku_name                 = "standard" # SKU tier for the Key Vault
  purge_protection_enabled = true       # Enables purge protection to prevent accidental deletion
}

# Set an access policy for the Key Vault to allow certain operations
resource "azurerm_key_vault_access_policy" "me" {
  key_vault_id = azurerm_key_vault.kv.id                      # Key Vault reference
  tenant_id    = data.azurerm_subscription.current.tenant_id  # Tenant ID
  object_id    = data.azurerm_client_config.current.object_id # Object ID of the principal

  key_permissions = [ # List of allowed key permissions
    "Create",
    "Get",
    "Delete",
    "Purge",
    "GetRotationPolicy",
  ]
}

# Create an Azure Storage Account
resource "azurerm_storage_account" "st" {
  name                     = var.storage_account_name           # Storage account name
  location                 = azurerm_resource_group.rg.location # Location from the resource group
  resource_group_name      = azurerm_resource_group.rg.name     # Resource group name
  account_tier             = "Standard"                         # Performance tier
  account_replication_type = "LRS"                              # Locally-redundant storage replication
}

# Deploy Azure AI Services resource
resource "azurerm_ai_services" "ai" {
  name                = var.ai_services_name               # AI Services resource name
  location            = azurerm_resource_group.rg.location # Location from the resource group
  resource_group_name = azurerm_resource_group.rg.name     # Resource group name
  sku_name            = "S0"                               # Pricing SKU tier
}

# Create GPT-4o deployment
resource "azurerm_cognitive_deployment" "gpt4o" {
  name                 = "gpt4o"
  cognitive_account_id = azurerm_ai_services.ai.id
  rai_policy_name      = "Microsoft.Default"
  model {
    format  = "OpenAI"
    name    = "gpt-4o"
    version = "2024-05-13"
  }
  sku {
    name     = "GlobalStandard"
    capacity = 200
  }
}

# Create Azure AI Foundry Hub
resource "azurerm_ai_foundry" "ai_foundry" {
  name                = var.ai_foundry_name             # AI Foundry service name
  location            = azurerm_ai_services.ai.location # Location from the AI Services resource
  resource_group_name = azurerm_resource_group.rg.name  # Resource group name
  storage_account_id  = azurerm_storage_account.st.id   # Associated storage account
  key_vault_id        = azurerm_key_vault.kv.id         # Associated Key Vault

  identity {
    type = "SystemAssigned" # Enable system-assigned managed identity
  }

  lifecycle {
    ignore_changes = [ # Ignore changes to the identity block
      tags,
    ]
  }
}

# Create an AI Foundry Project within the AI Foundry hub
resource "azurerm_ai_foundry_project" "ai_foundry_project" {
  name               = var.ai_foundry_project_name        # Project name
  location           = azurerm_resource_group.rg.location # Location from the AI Foundry hub
  ai_services_hub_id = azurerm_ai_foundry.ai_foundry.id   # Associated AI Foundry service

  identity {
    type = "SystemAssigned" # Enable system-assigned managed identity
  }
}

# Create a Bing Account resource using azapi
resource "azapi_resource" "bing_search" {
  type                      = "microsoft.bing/accounts@2020-06-10" # Resource type and API version
  name                      = var.bing_account_name                # Resource name
  location                  = "global"                             # Resource location
  parent_id                 = azurerm_resource_group.rg.id         # Parent resource group
  schema_validation_enabled = false
  body = {                  # Resource body
    kind = "Bing.Grounding" # Resource kind
    sku = {
      name = "G1" # SKU name
    }
  }
  response_export_values = ["properties.endpoint"]
}

# Use the azapi_resource to create a connection to AI Services
resource "azapi_resource" "ai_services_connection" {
  type                      = "Microsoft.MachineLearningServices/workspaces/connections@2025-01-01-preview" # Resource type and API version
  name                      = "${azurerm_ai_services.ai.name}-connection"                                   # Resource name
  location                  = "global"                                                                      # Resource location
  parent_id                 = azurerm_ai_foundry.ai_foundry.id                                              # Parent resource group
  schema_validation_enabled = false
  body = {
    properties = {
      category      = "AIServices"
      target        = azurerm_ai_services.ai.endpoint
      authType      = "ApiKey"
      isSharedToAll = true
      metadata = {
        ApiType    = "Azure"
        ResourceId = azurerm_ai_services.ai.id
      }
      credentials = {
        key = azurerm_ai_services.ai.primary_access_key
      }
    }
  }
}

# List keys for the Bing Account resource
data "azapi_resource_action" "bing_search" {
  type                   = "microsoft.bing/accounts@2020-06-10"
  resource_id            = azapi_resource.bing_search.id
  action                 = "listKeys"
  response_export_values = ["*"]
}

# Use the azapi_resource to create a connection to the Bing Account resource
resource "azapi_resource" "bing_connection" {
  type                      = "Microsoft.MachineLearningServices/workspaces/connections@2025-01-01-preview" # Resource type and API version
  name                      = "${azapi_resource.bing_search.name}-connection"                               # Resource name
  location                  = "global"                                                                      # Resource location
  parent_id                 = azurerm_ai_foundry.ai_foundry.id
  schema_validation_enabled = false
  body = {
    properties = {
      category      = "ApiKey"
      target        = azapi_resource.bing_search.output.properties.endpoint
      authType      = "ApiKey"
      isSharedToAll = true
      metadata = {
        Location   = "global"
        ApiType    = "Azure"
        ResourceId = azapi_resource.bing_search.id
      }
      credentials = {
        key = jsondecode(data.azapi_resource_action.bing_search.output).key1 # Use the key from the action result
      }
    }
  }
}

# Assign the "Azure AI Developer" role to the current user for the Foundry Project
resource "azurerm_role_assignment" "ai_foundry_project_developer" {
  scope                = azurerm_ai_foundry_project.ai_foundry_project.id
  role_definition_name = "Azure AI Developer"
  principal_id         = data.azurerm_client_config.current.object_id
}
```

### outputs.tf:

```bash
output "project_connection_string" {
  description = "The connection string to the AI Foundry project"
  value       = "${azurerm_ai_foundry_project.ai_foundry_project.location}.api.azureml.ms;${data.azurerm_subscription.current.subscription_id};${var.resource_group_name};${azurerm_ai_foundry_project.ai_foundry_project.name}"
}

output "deployment_name" {
  value = azurerm_cognitive_deployment.gpt4o.name
}

output "bing_connection_name" {
  description = "The connection name to the Bing Search resource"
  value       = azapi_resource.bing_connection.name
}
```

## Step 2: Initialize and Apply Terraform

Initialize Terraform and apply the configuration:

```bash
export ARM_SUBSCRIPTION_ID="<your-subscription-id>"
terraform init
terraform apply
```

Terraform will prompt for approval before creating resources. Review the plan and type `yes` to proceed.

## Step 3: What Does the Terraform Code Do?

The provided Terraform code:

- Configures the required providers (`azurerm`, `azapi`)
- Creates a resource group and supporting resources (Key Vault, Storage Account)
- Deploys Azure AI Services and a GPT-4o deployment
- Provisions Azure AI Foundry Hub and a Foundry Project
- Creates a Bing Account resource using the `azapi_resource`
- Establishes connections between Foundry and both Bing and AI Services (using API keys)
- Assigns the “Azure AI Developer” role to the current user for the Foundry Project

## Step 4: Outputs

After deployment, Terraform will output:

- The connection string to the AI Foundry project
- The name of the GPT-4o deployment
- The Bing Search connection name

You can use these outputs to configure your applications or notebooks.

## Conclusion

With just a few steps, you've deployed a fully functional Azure AI Foundry environment connected to Bing Search, all managed with Terraform. This approach ensures your infrastructure is reproducible, auditable, and easy to manage.

For the full code, check out the sample in [GitHub](https://github.com/cmendible/azure.samples/tree/main/ai_foundry_bing).

Hope it helps!

**References:**

- [Terraform Azure Provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [Terraform AzAPI Provider](https://registry.terraform.io/providers/Azure/azapi/latest/docs)
- [Azure AI Foundry documentation](https://learn.microsoft.com/en-us/azure/ai-services/foundry/)
