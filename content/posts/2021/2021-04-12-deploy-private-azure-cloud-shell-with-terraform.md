---
author: Carlos Mendible
categories:
- azure
crosspost_to_medium: true
date: "2021-04-12T10:00:00Z"
description: 'Deploy a Private Azure Cloud Shell with Terraform'
images: ["/assets/img/posts/azure.png"]
draft: false
tags: ["terraform", "cloud", "shell"]
title: 'Deploy a Private Azure Cloud Shell with Terraform'
---

By default Cloud Shell sessions run inside a container inside a Microsoft network separate from any resources you may have deployed in Azure. So what happens when you want to access services you have deployed inside a Virtual Network such as a private AKS cluster, a Virtual Machine or Private Endpoint enabled services? 

Well as you can imagine the solution is to deploy Cloud Shell into an Azure Virtual Network in such a way that the container, it runs on, can acces your private resources. This solution also protects the backing Storage Account that Cloud Shell uses for you the user profiles and data so that you end up with a locked down environmemt.

The following diagram shows the solution archictecture:

![Cloud Shell in an Azure Virtual Network Architecture Diagram](https://docs.microsoft.com/en-us/azure/cloud-shell/media/private-vnet/data-diagram.png)

If you need more information about the solution, the services and any limitations please check the documentation here: [Cloud Shell in an Azure Virtual Network](https://docs.microsoft.com/en-us/azure/cloud-shell/private-vnet). 

Now let's se how can we use terraform to make your Cloud Shell private! 

## Deploy a Private Azure Cloud Shell with Terraform

> If Cloud Shell has been used in the past, the existing clouddrive must be unmounted. To do this run `clouddrive unmount` from an active Cloud Shell session.

### 1. Create a `provider.tf` file with the following contents:

``` json
terraform {
  required_version = ">= 0.13.5"
}

provider "azurerm" {
  version = "= 2.46.1"
  features {}
}

provider "azuread" {
  version = "= 1.3.0"
}

provider "http" {
  version = "= 2.0.0"
}
```

Note that we are using `azurerm` to deploy Azure services, `azuread` to get some Service Principal information and `http` to get your current public ip address so that only you can reach your Cloud Shell. 

### 2. Create a variables.tf file with the following contents:

``` json
variable location {
  default = "west europe"
}

variable resource_group {
  default = "<resource group name>"
}

variable "vnet_name" {
  default = "<vnet name>"
}

variable sa_name {
  default = "<Backing Storage Account name>"
}

variable relay_name {
  default = "<Azure Relay name>"
}
```

Make sure you replace the placeholders with the default values you want to use.

### 3. Create a `main.tf` file with the following contents:

``` json
# Get Azure Container Instance Service Principal. Amazing right? Cloud Shell uses this Service Principal!
data "azuread_service_principal" "container" {
  display_name = "Azure Container Instance Service"
}

# Create Resource Groupto hold the resources.
resource "azurerm_resource_group" "rg" {
  name     = var.resource_group
  location = var.location
}

# Create a VNET.
resource "azurerm_virtual_network" "vnet" {
  name                = var.vnet_name
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

# Create a Containers Subnet. Here is where Cloud Shell will run.
resource "azurerm_subnet" "containers" {
  name                 = "cloudshell-containers"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.0.0/24"]

  # Delegate the subnet to "Microsoft.ContainerInstance/containerGroups".
  delegation {
    name = "cloudshell-delegation"

    service_delegation {
      name = "Microsoft.ContainerInstance/containerGroups"
    }
  }

  # Add service enpoint so Cloud Shell can reach Storage Accounts. At the moment the solution does not work with Private Enpoints for the Storage Account. 
  service_endpoints = ["Microsoft.Storage"]
}

# Create a subnet to host Azure Relay service.
resource "azurerm_subnet" "relay" {
  name                 = "relay"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.1.0/24"]

  enforce_private_link_endpoint_network_policies = true
}

# Create a network profile for the Cloud Shell containers.
resource "azurerm_network_profile" "networkprofile" {
  name                = "cloudshell"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  container_network_interface {
    name = "cloudshell-containers"

    ip_configuration {
      name      = "ipconfig"
      subnet_id = azurerm_subnet.containers.id
    }
  }
}

# Assign Network Contributor to the Azure Container Instance Service Principal.
resource "azurerm_role_assignment" "network_contributor" {
  scope                = azurerm_network_profile.networkprofile.id
  role_definition_name = "Network Contributor"
  principal_id         = data.azuread_service_principal.container.object_id
}

# Create an Azure Relay namespace.
resource "azurerm_relay_namespace" "relay" {
  name                = var.relay_name
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  sku_name = "Standard"
}

# Add a private enpoint to the Azure Relay namespace.
resource "azurerm_private_endpoint" "endpoint" {
  name                = "cloudshell-privateendpoint"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  subnet_id           = azurerm_subnet.relay.id

  private_service_connection {
    name                           = "privateendpoint"
    private_connection_resource_id = azurerm_relay_namespace.relay.id
    is_manual_connection           = false
    subresource_names              = ["namespace"]
  }
}

# Assign Contributor to the Azure Container Instance Service Principal.
resource "azurerm_role_assignment" "contributor" {
  scope                = azurerm_relay_namespace.relay.id
  role_definition_name = "Contributor"
  principal_id         = data.azuread_service_principal.container.object_id
}

# Create the Storage Account to hold the Cloud Shell profiles.
resource "azurerm_storage_account" "sa" {
  name                      = var.sa_name
  resource_group_name       = azurerm_resource_group.rg.name
  location                  = azurerm_resource_group.rg.location
  account_tier              = "Standard"
  account_replication_type  = "GRS"
  enable_https_traffic_only = true
}

# Create a file share to hold the user profiles.
resource "azurerm_storage_share" "share" {
  name                 = "profile"
  storage_account_name = azurerm_storage_account.sa.name
  quota                = 6
}

# Get your current public IP.
data "http" "current_public_ip" {
  url = "http://ipinfo.io/json"
  request_headers = {
    Accept = "application/json"
  }
}

# Protect the Storage Account setting the firewall.
# This is done only after the file share is created.
resource "azurerm_storage_account_network_rules" "sa_rules" {
  resource_group_name  = azurerm_resource_group.rg.name
  storage_account_name = azurerm_storage_account.sa.name

  default_action             = "Deny"
  virtual_network_subnet_ids = [azurerm_subnet.containers.id]

  # ip_rules = [
  #   jsondecode(data.http.current_public_ip.body).ip
  # ]

  depends_on = [
    azurerm_storage_share.share
  ]
}

# Create DNS Zone for Relay
resource "azurerm_private_dns_zone" "private" {
  name                = "privatelink.servicebus.windows.net"
  resource_group_name = azurerm_resource_group.rg.name
}

# Create A record for the Relay
resource "azurerm_private_dns_a_record" "relay" {
  name                = var.relay_name
  zone_name           = azurerm_private_dns_zone.private.name
  resource_group_name = azurerm_resource_group.rg.name
  ttl                 = 3600
  records             = [azurerm_private_endpoint.endpoint.private_service_connection[0].private_ip_address]
}

# Link the Private Zone with the VNet
resource "azurerm_private_dns_zone_virtual_network_link" "relay" {
  name                  = "relay"
  resource_group_name   = azurerm_resource_group.rg.name
  private_dns_zone_name = azurerm_private_dns_zone.private.name
  virtual_network_id    = azurerm_virtual_network.vnet.id
}

# Open the relay firewall to local IP
resource "null_resource" "open_relay_firewall" {
  provisioner "local-exec" {
    interpreter = ["powershell"]
    command = "az rest --method put --uri '${azurerm_relay_namespace.relay.id}/networkrulesets/default?api-version=2017-04-01' --body '{\"properties\":{\"defaultAction\":\\\"Deny\\\",\"ipRules\":[{\"ipMask\":\\\"${jsondecode(data.http.current_public_ip.body).ip}\\\"}],\"virtualNetworkRules\":[],\"trustedServiceAccessEnabled\":false}}'"
  }
  depends_on = [
    data.http.current_public_ip,
    azurerm_relay_namespace.relay
  ]
}
```

### 4. Deploy the solution:

Run the following commands:

``` shell
terraform init
terraform plan -out tf.plan
terraform apply ./tf.plan
```

Hope it helps! Please find the complete code [here](https://github.com/cmendible/azure.samples/tree/main/private.cloud.shell)