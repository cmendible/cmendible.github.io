---
author: Carlos Mendible
categories:
- azure
crosspost_to_medium: true
date: "2021-09-17T10:00:00Z"
description: 'Static website hosting in an Azure Storage Account protected with Private Endpoint'
images: ["/assets/img/posts/azure.png"]
draft: false
tags: ["static website", "storage account", "private endpoint"]
title: 'Static website hosting in an Azure Storage Account protected with Private Endpoint'
---

This post will show you how to deploy a Static Website on a Storage Account protected with Private Endpoint using Terraform:

## Define the terraform providers to use

Create a *providers.tf* file with the following contents:

``` yaml
terraform {
  required_version = "> 0.12"
  required_providers {
    azurerm = {
      source  = "azurerm"
      version = "~> 2.26"
    }
  }
}

provider "azurerm" {
  features {}
  skip_provider_registration = true
}
```

## Define the variables

Create a *variables.tf* file with the following contents:

``` yaml
variable "location" {
  default = "west europe"
}

variable "resource_group" {
  default = "web-sta-private-endpoint"
}

variable "sa_name" {
  default = "webstapecfm"
}
```

## Define the required resources

Create a *main.tf* file with the following contents:

``` yaml
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
}

# Create the Subnet for the jumpbox.
resource "azurerm_subnet" "jump" {
  name                 = "jump"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.1.0/24"]
}

# Create the Subnet for the private endpoints. This is where the IP of the private endpoint will live.
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

  static_website {
  }
}

resource "azurerm_storage_blob" "page" {
  name                   = "index.html"
  storage_account_name   = azurerm_storage_account.sa.name
  storage_container_name = "$web"
  type                   = "Block"
  source                 = "index.html"
}

# Create the privatelink.web.core.windows.net Private DNS Zone
resource "azurerm_private_dns_zone" "private_web" {
  name                = "privatelink.web.core.windows.net"
  resource_group_name = azurerm_resource_group.rg.name
}

# Create the Private endpoint. This is where the Storage account gets a private IP inside the VNet.
resource "azurerm_private_endpoint" "endpoint_web" {
  name                = "sa-endpoint_web"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  subnet_id           = azurerm_subnet.endpoint.id

  private_service_connection {
    name                           = "sa-privateserviceconnection-web"
    private_connection_resource_id = azurerm_storage_account.sa.id
    is_manual_connection           = false
    subresource_names              = ["web"]
  }

  private_dns_zone_group {
    name                 = "privatelink-file-web-core-windows-net"
    private_dns_zone_ids = [azurerm_private_dns_zone.private_web.id]
  }
}

# Link the Private Zone with the VNet
resource "azurerm_private_dns_zone_virtual_network_link" "sa_private_web" {
  name                  = "test_web"
  resource_group_name   = azurerm_resource_group.rg.name
  private_dns_zone_name = azurerm_private_dns_zone.private_web.name
  virtual_network_id    = azurerm_virtual_network.vnet.id
}

# Public IP for the jumpbox
resource "azurerm_public_ip" "pip" {
  name                = "jumpbox-ip"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method   = "Static"
}

# NIC for the jumpbox
resource "azurerm_network_interface" "nic" {
  name                = "jumpbox-nic"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.jump.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.pip.id
  }
}

# Create the jumpbox VM
resource "azurerm_linux_virtual_machine" "jumpbox" {
  name                = "jumpbox"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_F2"
  admin_username      = "adminuser"
  network_interface_ids = [
    azurerm_network_interface.nic.id,
  ]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  admin_ssh_key {
    username   = "adminuser"
    public_key = file(pathexpand("~/.ssh/id_rsa.pub"))
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "UbuntuServer"
    sku       = "16.04-LTS"
    version   = "latest"
  }
}
```

**Note:** 

* The above definition will deploy a jumpbox server with a public IP address so you can access it from your PC. This jumpbox is also deployed in the same VNet as the Storage Account's Private Endpoint.
* The definition requires your public key to be in `~/.ssh/id_rsa.pub`

## Define the outputs

Create a *outputs.tf* file with the following contents:

``` yaml
output "jumpbox_ip" {
  value = azurerm_public_ip.pip.ip_address
}

output "sa_name" {
  value = var.sa_name
}
```

## Create a simple Web Page

Create a *index.html* file with the following contents:

``` html
<!doctype html>
<html>
  <head>
    <title>This is your private static web app.</title>
  </head>
  <body>
    <p>This is your private static web app.</p>
  </body>
</html>
```

## Deploy the resources

Run:

``` shell
terraform init
terraform apply
```

## Test the static website

Login to the jumpbox server 

``` shell
$jumpboxIp=$(terraform output jumpbox_ip)
ssh adminuser@$jumpboxIp
```

Run the following command, to test the static website:

``` shell
curl -i https://<storage account name>.z6.web.core.windows.net/index.html
```

You should get a response similar to:

``` shell
HTTP/1.1 200 OK
Content-Length: 180
Content-Type: application/octet-stream
Last-Modified: Fri, 17 Sep 2021 11:59:00 GMT
Accept-Ranges: bytes
ETag: "0x8D979D28981C913"
Server: Windows-Azure-Web/1.0 Microsoft-HTTPAPI/2.0
x-ms-request-id: fb6880b7-a01e-0014-42bb-ab0938000000
x-ms-version: 2018-03-28
Date: Fri, 17 Sep 2021 12:01:41 GMT

<!doctype html>
<html>
  <head>
    <title>This is your private static web app.</title>
  </head>
  <body>
    <p>This is your private static web app.</p>
  </body>
</html>
```

That's it. Hope it helps!!!

Please find the complete **terraform** configuration [here](https://github.com/cmendible/azure.samples/tree/main/static_website_storage_account_private_endpoint/deploy)