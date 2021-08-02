---
author: Carlos Mendible
categories:
- kubernetes
- azure
crosspost_to_medium: true
date: "2021-08-02T10:00:00Z"
description: 'AKS: Persistent Volume Claim with an Azure File Storage protected with a Private Endpoint'
images: ["/assets/img/posts/aks.png"]
draft: false
tags: ["csi", "pvc", "azurefiles"]
title: 'AKS: Persistent Volume Claim with an Azure File Storage protected with a Private Endpoint'
---

This post will show you the steps you'll have to take to deploy an Azure Files Storage with a Private Endpoint and use it to create volumes for an Azure Kubernetes Service cluster:

## Create a **bicep** file to declare the Azure resources

You'll have to declare the following resources:

* A VNET with 2 subnets. One for the private endpoint and the other for the AKS cluster.
* An Azure Files storage.
* A Private Endpoint for the storage.
* A Private DNS Zone and Private DNS Zone Group.
* A link between the Private DNS Zone to the VNET.
* An AKS cluster.
* A role assignment to add the kubelet identity of the cluster as a Contributor to the Storage Account.

in a *main.bicep* file with the following contents:

``` json
param sa_name string = 'akscsisa'
param aks_name string = 'akscsimsft'

// Create the VNET
resource vnet 'Microsoft.Network/virtualNetworks@2020-11-01' = {
  name: 'private-network'
  location: 'westeurope'
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.0.0.0/8'
      ]
    }
    subnets: [
      {
        name: 'endpoint'
        properties: {
          addressPrefix: '10.241.0.0/16'
          serviceEndpoints: []
          delegations: []
          privateEndpointNetworkPolicies: 'Disabled'
          privateLinkServiceNetworkPolicies: 'Enabled'
        }
      }
      {
        name: 'aks'
        properties: {
          addressPrefix: '10.240.0.0/16'
          serviceEndpoints: []
          delegations: []
          privateEndpointNetworkPolicies: 'Enabled'
          privateLinkServiceNetworkPolicies: 'Enabled'
        }
      }
    ]
    enableDdosProtection: false
  }
}

// Create the File Storage Account
resource sa 'Microsoft.Storage/storageAccounts@2021-01-01' = {
  name: sa_name
  location: 'westeurope'
  sku: {
    name: 'Premium_LRS'
    tier: 'Premium'
  }
  kind: 'FileStorage'
  properties: {
    minimumTlsVersion: 'TLS1_0'
    allowBlobPublicAccess: false
    isHnsEnabled: false
    networkAcls: {
      bypass: 'AzureServices'
      virtualNetworkRules: []
      ipRules: []
      defaultAction: 'Deny'
    }
    supportsHttpsTrafficOnly: true
    accessTier: 'Hot'
  }
}

// Create the Private Enpoint
resource private_endpoint 'Microsoft.Network/privateEndpoints@2020-11-01' = {
  name: 'sa-endpoint'
  location: 'westeurope'
  properties: {
    privateLinkServiceConnections: [
      {
        name: 'sa-privateserviceconnection'
        properties: {
          privateLinkServiceId: sa.id
          groupIds: [
            'file'
          ]
        }
      }
    ]
    subnet: {
      id: '${vnet.id}/subnets/endpoint'
    }
  }
}

// Create the Private DNS Zone
resource dns 'Microsoft.Network/privateDnsZones@2018-09-01' = {
  name: 'privatelink.file.core.windows.net'
  location: 'global'
}

// Link the Private DNS Zone with the VNET
resource vnet_dns_link 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2018-09-01' = {
  name: '${dns.name}/test'
  location: 'global'
  properties: {
    registrationEnabled: false
    virtualNetwork: {
      id: vnet.id
    }
  }
}

// Create Private DNS Zone Group 
resource dns_group 'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2020-11-01' = {
  name: '${private_endpoint.name}/default'
  properties: {
    privateDnsZoneConfigs: [
      {
        name: 'privatelink-file-core-windows-net'
        properties: {
          privateDnsZoneId: dns.id
        }
      }
    ]
  }
}

// Create AKS cluster
resource aks 'Microsoft.ContainerService/managedClusters@2021-02-01' = {
  name: aks_name
  location: 'westeurope'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    kubernetesVersion: '1.19.9'
    dnsPrefix: aks_name
    agentPoolProfiles: [
      {
        name: 'default'
        count: 1
        vmSize: 'Standard_D2s_v3'
        osDiskSizeGB: 30
        osDiskType: 'Ephemeral'
        vnetSubnetID: '${vnet.id}/subnets/aks'
        type: 'VirtualMachineScaleSets'
        orchestratorVersion: '1.19.9'
        osType: 'Linux'
        mode: 'System'
      }
    ]
    servicePrincipalProfile: {
      clientId: 'msi'
    }
    addonProfiles: {
      kubeDashboard: {
        enabled: false
      }
    }
    enableRBAC: true
    networkProfile: {
      networkPlugin: 'kubenet'
      networkPolicy: 'calico'
      loadBalancerSku: 'standard'
      podCidr: '10.244.0.0/16'
      serviceCidr: '10.0.0.0/16'
      dnsServiceIP: '10.0.0.10'
      dockerBridgeCidr: '172.17.0.1/16'
    }
    apiServerAccessProfile: {
      enablePrivateCluster: false
    }
  }
}

// Built-in Role Definition IDs
// https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles
var contributor = '/subscriptions/${subscription().subscriptionId}/providers/Microsoft.Authorization/roleDefinitions/b24988ac-6180-42a0-ab88-20f7382dd24c'

// Set AKS kubelet Identity as SA Contributor
resource aks_kubelet_sa_contributor 'Microsoft.Authorization/roleAssignments@2020-04-01-preview' = {
  name: guid('${aks_name}_kubelet_sa_contributor')
  scope: sa
  properties: {
    principalId: reference(aks.id, '2021-02-01', 'Full').properties.identityProfile['kubeletidentity'].objectId
    roleDefinitionId: contributor
  }
}
```

## Deploy the Azure resources

Run the following commands to deploy the Azure resources:

``` bash
az group create -n private-pvc-test -l westeurope

az deployment group create -f ./main.bicep -g private-pvc-test
```

After 10 minutes or so you'll have all resources up and running.

## Install Azure CSI Driver

The [Azure Files Container Storage Interface (CSI) driver](https://github.com/kubernetes-sigs/azurefile-csi-driver) can be installed on the cluster so Azure Kubernetes Service (AKS) can manage the lifecycle of Azure Files shares.

To install the driver run:

``` bash
az aks get-credentials -n akscsimsft -g private-pvc-test --overwrite-existing

curl -skSL https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/v1.5.0/deploy/install-driver.sh | bash -s v1.5.0 --
```

and check the pods status:

``` bash
kubectl -n kube-system get pod -o wide --watch -l app=csi-azurefile-controller
kubectl -n kube-system get pod -o wide --watch -l app=csi-azurefile-node
```

You should find instances of *csi-azurefile-controller* and *csi-azurefile-node* running without issues.

## Create a Storage Class

Create a file named: *storageclass-azurefile-csi.yaml* with the following yaml:

``` yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azurefile-csi
provisioner: file.csi.azure.com
allowVolumeExpansion: true
parameters:
  resourceGroup: <resourceGroup>  # optional, only set this when storage account is not in the same resource group as agent node
  storageAccount: <storageAccountName>
  # Check driver parameters here:
  # https://github.com/kubernetes-sigs/azurefile-csi-driver/blob/master/docs/driver-parameters.md
  server: <storageAccountName>.privatelink.file.core.windows.net 
reclaimPolicy: Delete
volumeBindingMode: Immediate
mountOptions:
  - dir_mode=0777
  - file_mode=0777
  - uid=0
  - gid=0
  - mfsymlinks
  - cache=strict  # https://linux.die.net/man/8/mount.cifs
  - nosharesock  # reduce probability of reconnect race
  - actimeo=30  # reduce latency for metadata-heavy workload
```

Remember to replace the values of `<resourceGroup>` and `<storageAccountName>` with the ones used in the previous deployment. (i.e. *private-pvc-test* and *akscsisa*)

Now deploy the Storage Class to the cluster:

``` bash
kubectl apply -f storageclass-azurefile-csi.yaml
```

## Create a Private Volume Claim

Create a Private Volume Claim that uses the storage class. Create *pvc.yaml* with the following contents:

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-azurefile
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azurefile-csi
  resources:
    requests:
      storage: 100Gi
```

Deploy the Private Volume Claim to the cluster and check that everything is ok:

``` bash
kubectl apply -f pvc.yaml
k get pvc
```

Now feel free to try and mount a volume using the Private Volume Claim: *my-azurefile*  

Hope it helps!!!

Please find a **bicep** based sample [here](https://github.com/cmendible/azure.samples/tree/main/aks_csi_sa_private_enpoint/bicep) or if you prefer **terraform** [here](https://github.com/cmendible/azure.samples/tree/main/aks_csi_sa_private_enpoint/terraform)

References:
* [Use Azure Files Container Storage Interface (CSI) drivers in Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/azure-files-csi)
* [Azure Files Container Storage Interface (CSI) driver](https://github.com/kubernetes-sigs/azurefile-csi-driver) 