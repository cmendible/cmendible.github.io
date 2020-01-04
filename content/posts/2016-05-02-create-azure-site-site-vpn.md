---
author: Carlos Mendible
categories:
- Azure
- DevOps
date: "2016-05-02T18:30:24Z"
description: Create an Azure Site to Site VPN
image: /wp-content/uploads/2016/05/Cloud-Azure.jpg
# tags: HybridCloud PowerShell VPN Site-to-Site
title: Create an Azure Site to Site VPN
---
In this post I'll just show the list of PowerShell commands needed to **Create an Azure Site to Site VPN** and give you some tips when using a <a href="http://www.checkpoint.com/" target="_blank">Check Point Security Gateway</a>.

First things first! so if you have not installed and configured Azure PowerShell go for it: <a href="https://azure.microsoft.com/en-us/documentation/articles/powershell-install-configure/" target="_blank">How to install and configure Azure PowerShell</a>

## 1. Connect and Select you subscription
---
Start by logging into Azure and selecting you subscription:
    
``` powershell
Login-AzureRmAccount
Get-AzureRmSubscription
Select-AzureRmSubscription -SubscriptionId xxxxxxxx-xxx-xxxx-xxxx-xxxxxxxxxxx
```

## 2. Create a Resource Group
---
Create a resource group. Let's name it: **resourceGroup**
    
``` powershell
New-AzureRmResourceGroup -Name resourceGroup -Location 'West Europe'
```

## 3. Create a virtual network and a gateway subnet
---
It's necessary to create at least two subnets, and one must be named **GatewaySubnet** for everything to work as expected.

Give the other subnet the name: **remoteSubnet**. This is where your Virtual Machines will live.

In this case both subnets are contained in the 10.0.100.0/24 address space.

``` powershell
$subnet1 = New-AzureRmVirtualNetworkSubnetConfig -Name 'GatewaySubnet' -AddressPrefix 10.0.100.0/25
$subnet2 = New-AzureRmVirtualNetworkSubnetConfig -Name 'remoteSubnet' -AddressPrefix 10.0.100.128/25
New-AzureRmVirtualNetwork -Name cloudNetwork -ResourceGroupName resourceGroup -Location 'West Europe' -AddressPrefix 10.0.100.0/24 -Subnet $subnet1, $subnet2
```

## 4. Add your local network gateway
---
Here you must specify the IP address of your on-premises VPN device (i.e 95.122.110.101) which cannot be located behind a NAT.

You must also specify your on-premises address space that you want to reach from the cloud. In this sample we will reach anything in the following ranges: 10.2.0.0/21 and &#8216;192.168.106.0/24'

    
``` powershell
New-AzureRmLocalNetworkGateway -Name LocalSite -ResourceGroupName resourceGroup -Location 'West Europe' -GatewayIpAddress '95.122.110.101' -AddressPrefixÂ  @('10.2.0.0/21','192.168.106.0/24')
```

## 5. Request a public IP address for the VPN gateway
---
Next request a public IP Address and give it a name like **globalIP**
    
``` powershell
$gwpip= New-AzureRmPublicIpAddress -Name globalIP -ResourceGroupName resourceGroup -Location 'West Europe' -AllocationMethod Dynamic
```

## 6. Create the gateway IP addressing configuration
---

``` powershell
$vnet = Get-AzureRmVirtualNetwork -Name cloudNetwork -ResourceGroupName resourceGroup
$subnet = Get-AzureRmVirtualNetworkSubnetConfig -Name 'GatewaySubnet' -VirtualNetwork $vnet
$gwipconfig = New-AzureRmVirtualNetworkGatewayIpConfig -Name gwipconfig -SubnetId $subnet.Id -PublicIpAddressId $gwpip.Id
```

## 7. Create the virtual network gateway
---
Sit back! this step can take up to 10 or more minutes while your Virtual Network Gateway is created.
    
``` powershell
New-AzureRmVirtualNetworkGateway -Name vnetgw -ResourceGroupName resourceGroup -Location 'West Europe' -IpConfigurations $gwipconfig -GatewayType Vpn -VpnType RouteBased -GatewaySku Standard
```

## 8. Get your VPN Public IP Address
---
Now you can get the cloud public IP address you must configure in your on-premises VPN device.
    
``` powershell
Get-AzureRmPublicIpAddress -Name gwpip -ResourceGroupName resourceGroup
```

## 9. Configure your VPN device
For our Check Point device we followed the:<a href="https://supportcenter.checkpoint.com/supportcenter/portal?eventSubmit_doGoviewsolutiondetails=&#038;solutionid=sk101275" target="_blank"> How to setup Site-to-Site VPN between Check Point and Microsoft Azure guide</a>
    
Take note that the commands given here will result in a **Gateway to Gateway** Azure setup and therefor you'll have to configure **One VPN tunnel per Gateway pair** in your Check Point device.

Also save the Shared Key (i.e secretword) you'll need it in the next step.
           
## 10. Create the VPN connection
---
It's time to create your connection (named localtoazure in the sample) and your **Azure Site-to-Site VPN** will start working in a snap!
          
``` powershell
$gateway1 = Get-AzureRmVirtualNetworkGateway -Name vnetgw -ResourceGroupName resourceGroup
$local = Get-AzureRmLocalNetworkGateway -Name LocalSite -ResourceGroupName resourceGroup
New-AzureRmVirtualNetworkGatewayConnection -Name localtoazure -ResourceGroupName resourceGroup -Location 'West Europe' -VirtualNetworkGateway1 $gateway1 -LocalNetworkGateway2 $local -ConnectionType IPsec -RoutingWeight 10 -SharedKey 'secretword'
```

Hope it helps!
  
References:

<ul>
  <li>
    <a href="https://azure.microsoft.com/en-us/documentation/articles/vpn-gateway-create-site-to-site-rm-powershell/" target="_blank">Create a virtual network with a Site-to-Site VPN connection using PowerShell and Azure Resource Manager</a>
  </li>
  <li>
    <a href="https://azure.microsoft.com/en-us/documentation/articles/vpn-gateway-about-vpn-devices/" target="_blank">About VPN devices for Site-to-Site VPN Gateway connections</a>
  </li>
  <li>
    <a href="https://supportcenter.checkpoint.com/supportcenter/portal?eventSubmit_doGoviewsolutiondetails=&solutionid=sk101275" target="_blank">How to setup Site-to-Site VPN between Check Point and Microsoft Azure</a>
  </li>
</ul>