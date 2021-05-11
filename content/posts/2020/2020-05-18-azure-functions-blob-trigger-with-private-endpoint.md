---
author: Carlos Mendible
categories:
- azure
crosspost_to_medium: true
date: "2020-05-18T00:00:00Z"
description: 'Azure Functions: use Blob Trigger with Private Endpoint'
images: ["/assets/img/posts/azure.png"]
published: true
tags: ["functions", "serverless", "blob", "privateendpoint"]
title: 'Azure Functions: use Blob Trigger with Private Endpoint'
---

The intent of this post is to help you understand how to connect an Azure Function to a Storage Account privately so all traffic flows through a VNet therefore enhancing the security of your solutions and blobs.

## The Case:

Supose you have the following Azure Function written in C# which only copies a blob from one conatiner to another:

```csharp
using System.IO;
using Microsoft.Azure.WebJobs;
using Microsoft.Extensions.Logging;

namespace Secured.Function
{
    public static class SecureCopy
    {
        [FunctionName("SecureCopy")]
        public static void Run(
        [BlobTrigger("input/{name}", Connection = "privatecfm_STORAGE")] Stream myBlob,
        [Blob("output/{name}", FileAccess.Write, Connection = "privatecfm_STORAGE")] Stream copy,
        ILogger log)
        {
            myBlob.CopyTo(copy);
        }
    }
}
```

In this case the storage account used, for the blob trigger and the output binding, has a public endpoint exposed to the internet, which you can secure using features such as the Storage Account Firewall and the new **private endpoints** which will allow clients on a virtual network (VNet) to securely access data over a [Private Link](https://docs.microsoft.com/en-us/azure/private-link/private-link-overview). The private endpoint uses an IP address from the VNet address space for your storage account service. [[1]](#references)

With those features we can lockdown all inbound traffic to the Storage Account to only accept calls from inside a VNet, so the next step is to enable a feature for Azure Functions that will give your function app access to the resources in the VNet: **Azure App Service VNet Integration feature** [[2]](#references).

The following sketch shows how this works:

![Solution Diagram](/assets/img/posts/function_private_endpoint_sa.png)

1. The Azure Function is integrated with a VNet using Regional VNet Integration (blue line).
1. The Storage Account (shown on the right) has a Private Endpoint which assigns a private IP to the Storage Account.
1. Traffic (red line) from the Azure Function flows through the VNet, the Private Endpoint and reaches the Storage Account.
1. The Storage Account, shown on the left, is used for the core services of the Azure Function and, ~~at the time of writing, can't be protected using private enpoints~~. [Check Update 2020-08-25](#update-2020-08-25).

But wait there is one more thing, you will need to add an Azure Private DNS Zone to enable the Azure Function to resolve the name of the Storage Account so it uses the private ip for communication.)

**Note:** The solution will require use of the PremiumV2, or Elastic Premium pricing plan for the Azure Function.

## Deploying the Infrastructure with Terraform

We'll be using Terraform (version > 0.12)  to deploy the solution. Start creating a **providers.tf** file with the following contents:

```yaml
terraform {
  required_version = "> 0.12"
}

provider "azurerm" {
  version = ">= 2.0"
  features {}
}
```

Define the following variables in a **variables.tf** file:

```yaml
# Azure Resource Location
variable location {
  default = "west europe"
}

# Azure Resource Group Name
variable resource_group {
  default = "private-endpoint"
}

# Name of the Storage Account you'll expose through the private endpoint
variable sa_name {
  default = "privatecfm"
}

# Name of the Storage Account backing the Azure Function
variable function_required_sa {
  default = "privatecfmfunc"
}
```

Create a **mainty.tf** file with the following contents (Make sure you read the comments to understand the manifest):

```yaml
# Create Resource Group
resource "azurerm_resource_group" "rg" {
  name     = var.resource_group
  location = var.location
}

# Create VNet
resource "azurerm_virtual_network" "vnet" {
  name                = "private-network"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  # Use Private DNS Zone. That's right we have to add this magical IP here.
  # dns_servers = ["168.63.129.16"]
}

# Create the Subnet for the Azure Function. This is thge subnet where we'll enable Vnet Integration.
resource "azurerm_subnet" "service" {
  name                 = "service"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.1.0/24"]

  enforce_private_link_service_network_policies = true

  # Delegate the subnet to "Microsoft.Web/serverFarms"
  delegation {
    name = "acctestdelegation"

    service_delegation {
      name    = "Microsoft.Web/serverFarms"
      actions = ["Microsoft.Network/virtualNetworks/subnets/action"]
    }
  }
}

# Create the Subnet for the private endpoints. This is where the IP of the private enpoint will live.
resource "azurerm_subnet" "endpoint" {
  name                 = "endpoint"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.2.0/24"]

  enforce_private_link_endpoint_network_policies = true
}

# Get current public IP. We'll need this so we can access the Storage Account from our PC.
data "http" "current_public_ip" {
  url = "http://ipinfo.io/json"
  request_headers = {
    Accept = "application/json"
  }
}

# Create the "private" Storage Account.
resource "azurerm_storage_account" "sa" {
  name                      = var.sa_name
  resource_group_name       = azurerm_resource_group.rg.name
  location                  = azurerm_resource_group.rg.location
  account_tier              = "Standard"
  account_replication_type  = "GRS"
  enable_https_traffic_only = true
  # We are enabling the firewall only allowing traffic from our PC's public IP.
  network_rules {
    default_action             = "Deny"
    virtual_network_subnet_ids = []
    ip_rules = [
      jsondecode(data.http.current_public_ip.body).ip
    ]
  }
}

# Create input container
resource "azurerm_storage_container" "input" {
  name                  = "input"
  container_access_type = "private"
  storage_account_name  = azurerm_storage_account.sa.name
}

# Create output container
resource "azurerm_storage_container" "output" {
  name                  = "output"
  container_access_type = "private"
  storage_account_name  = azurerm_storage_account.sa.name
}

# Create the Private endpoint. This is where the Storage account gets a private IP inside the VNet.
resource "azurerm_private_endpoint" "endpoint" {
  name                = "sa-endpoint"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  subnet_id           = azurerm_subnet.endpoint.id

  private_service_connection {
    name                           = "sa-privateserviceconnection"
    private_connection_resource_id = azurerm_storage_account.sa.id
    is_manual_connection           = false
    subresource_names              = ["blob"]
  }
}

# Create the blob.core.windows.net Private DNS Zone
resource "azurerm_private_dns_zone" "private" {
  name                = "privatelink.blob.core.windows.net"
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_private_dns_cname_record" "cname" {
  name                = "${var.sa_name}.blob.core.windows.net"
  zone_name           = azurerm_private_dns_zone.private.name
  resource_group_name = azurerm_resource_group.rg.name
  ttl                 = 300
  record              = "${var.sa_name}.privatelink.blob.core.windows.net"
}

# Create an A record pointing to the Storage Account private endpoint
resource "azurerm_private_dns_a_record" "sa" {
  name                = var.sa_name
  zone_name           = azurerm_private_dns_zone.private.name
  resource_group_name = azurerm_resource_group.rg.name
  ttl                 = 3600
  records             = [azurerm_private_endpoint.endpoint.private_service_connection[0].private_ip_address]
}

# Link the Private Zone with the VNet
resource "azurerm_private_dns_zone_virtual_network_link" "sa" {
  name                  = "test"
  resource_group_name   = azurerm_resource_group.rg.name
  private_dns_zone_name = azurerm_private_dns_zone.private.name
  virtual_network_id    = azurerm_virtual_network.vnet.id
}

# Create the Storage Account required by Azure Functions.
resource "azurerm_storage_account" "function_required_sa" {
  name                      = var.function_required_sa
  resource_group_name       = azurerm_resource_group.rg.name
  location                  = azurerm_resource_group.rg.location
  account_tier              = "Standard"
  account_replication_type  = "GRS"
  enable_https_traffic_only = true
}

# Create a container to hold the Azure Function Zip
resource "azurerm_storage_container" "functions" {
  name                  = "function-releases"
  storage_account_name  = azurerm_storage_account.function_required_sa.name
  container_access_type = "private"
}

# Create a blob with the Azure Function zip
resource "azurerm_storage_blob" "function" {
  name                   = "securecopy.zip"
  storage_account_name   = azurerm_storage_account.function_required_sa.name
  storage_container_name = azurerm_storage_container.functions.name
  type                   = "Block"
  source                 = "./securecopy.zip"
}

# Create a SAS token so the Function can access the blob and deploy the zip
data "azurerm_storage_account_sas" "sas" {
  connection_string = azurerm_storage_account.function_required_sa.primary_connection_string
  https_only        = false
  resource_types {
    service   = false
    container = false
    object    = true
  }
  services {
    blob  = true
    queue = false
    table = false
    file  = false
  }
  start  = "2020-05-18"
  expiry = "2025-05-18"
  permissions {
    read    = true
    write   = false
    delete  = false
    list    = false
    add     = false
    create  = false
    update  = false
    process = false
  }
}

# Create the Azure Function plan (Elastic Premium) 
resource "azurerm_app_service_plan" "plan" {
  name                = "azure-functions-test-service-plan"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  kind = "elastic"
  sku {
    tier     = "ElasticPremium"
    size     = "EP1"
    capacity = 1
  }
}

# Create Application Insights
resource "azurerm_application_insights" "ai" {
  name                = "func-pe-test"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  application_type    = "web"
  retention_in_days   = 90
}

# Create the Azure Function App
resource "azurerm_function_app" "func_app" {
  name                       = "func-pe-test"
  location                   = azurerm_resource_group.rg.location
  resource_group_name        = azurerm_resource_group.rg.name
  app_service_plan_id        = azurerm_app_service_plan.plan.id
  storage_account_name       = azurerm_storage_account.function_required_sa.name
  storage_account_access_key = azurerm_storage_account.function_required_sa.primary_access_key
  version                    = "~3"

  app_settings = {
    https_only                     = true
    APPINSIGHTS_INSTRUMENTATIONKEY = azurerm_application_insights.ai.instrumentation_key
    privatecfm_STORAGE             = azurerm_storage_account.sa.primary_connection_string
    # With this setting we'll force all outbound traffic through the VNet
    WEBSITE_VNET_ROUTE_ALL = "1"
    WEBSITE_DNS_SERVER     = "168.63.129.16"
    # Properties used to deploy the zip
    HASH            = filesha256("./securecopy.zip")
    WEBSITE_USE_ZIP = "https://${azurerm_storage_account.function_required_sa.name}.blob.core.windows.net/${azurerm_storage_container.functions.name}/${azurerm_storage_blob.function.name}${data.azurerm_storage_account_sas.sas.sas}"
  }
}

# Enable Regional VNet integration. Function --> service Subnet 
resource "azurerm_app_service_virtual_network_swift_connection" "vnet_integration" {
  app_service_id = azurerm_function_app.func_app.id
  subnet_id      = azurerm_subnet.service.id
}
```

Now download, into your working folder, the **[securecopy.zip](https://github.com/cmendible/azure.samples/blob/master/function_sa_private_endpoint/deploy/securecopy.zip)** file containing the sample Azure Function or create a zip, with the same name, containing your own code. 

Deploy the solution running the following commands:

```shell
terraform init
terraform apply
```

## Test the solution

Use the Azure portal or Storage Explorer to upload a file to the **input** container, after a few seconds you should find a copy of the file in the **output** container.

## VNET Integration Name Resolution Test

If the previous test didn't work, please connect through KUDU or the Console to the Azure Function and run the following command:

```shell
nameresolver <name of the storage account>.blob.core.windows.net
```

The output of the command should show **10.0.2.4** as the IP address. If that's not the case you probably misconfigured something. [[3]](#references)

Hope it helps and please find a copy of all the code [here](https://github.com/cmendible/azure.samples/tree/main/function_sa_private_endpoint)

## Update 2020-08-25

I've added a new sample [here](https://github.com/cmendible/azure.samples/tree/main/function_sa_private_endpoint.v2) using the same Storage Account for both: backing the Azure Function and the Blobs needed for the sample application.

With that script you must run Terraform apply twice: the first time with the Storage Account Firewall disabled so the Azure Function deployment runs without any issues and the second one enabling the Firewall so the Storage Account is protected.

> This setup requires 4 Private Enpoints: one for each of the Storage Account Services required by the Azure Function runtime (blob. table, queue, files).

```shell
terraform apply -var="sa_firewall_enabled=false"
terraform apply -var="sa_firewall_enabled=true"
```

## References

* [1] [Use private endpoints for Azure Storage](https://docs.microsoft.com/en-us/azure/storage/common/storage-private-endpoints?WT.mc_id=AZ-MVP-5002618)
* [2] Integrate your app with an Azure virtual network: [Regional VNET Integration](https://docs.microsoft.com/en-us/azure/app-service/web-sites-integrate-with-vnet#regional-vnet-integration?WT.mc_id=AZ-MVP-5002618)
* [3] Integrate your app with an Azure virtual network: [Troubleshooting](https://docs.microsoft.com/en-us/azure/storage/common/storage-private-endpoints?WT.mc_id=AZ-MVP-5002618)
