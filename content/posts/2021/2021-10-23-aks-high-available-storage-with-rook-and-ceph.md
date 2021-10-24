---
author: Carlos Mendible
categories:
- azure
- kubernetes
crosspost_to_medium: false
date: "2021-10-23T10:00:00Z"
description: 'AKS: High Available Storage with Rook and Ceph'
images: ["/assets/img/posts/aks.png"]
draft: false
tags: ["rook", "ceph", "storage"]
title: 'AKS: High Available Storage with Rook and Ceph'
---

> Disclaimer: this is just a Proof of Concept.

If you deploy Azure Kubernetes Service clusters with availability zones, you'll probaly need a high available storage solution.

In such situation you may use Azure Files as an external storage solution. But what if you need something that performs better? Or something running inside your cluster?

Well let me present you [Rook](https://rook.io) 

> "Rook turns distributed storage systems into self-managing, self-scaling, self-healing storage services. It automates the tasks of a storage administrator: deployment, bootstrapping, configuration, provisioning, scaling, upgrading, migration, disaster recovery, monitoring, and resource management."

and [Ceph](https://ceph.io).

> "Ceph is an open-source software (software-defined storage) storage platform, implements object storage on a single distributed computer cluster, and provides 3-in-1 interfaces for object-, block- and file-level storage."

Combining these two technologies, Rook and Ceph, we can create a available storage solution using Kubernetes tools such as helm and primitives such as PVCs.

Let me show you how to deploy Rook and Ceph on Azure Kubernetes Service:

## Deploy cluster with Rook and Ceph using Terraform

### Create variables.tf with the following contents:

``` terraform
# Location of the services
variable "location" {
  default = "west europe"
}

# Resource Group Name
variable "resource_group" {
  default = "aks-rook"
}

# Name of the AKS cluster
variable "aks_name" {
  default = "aks-rook"
}
```

### Create provider.tf with the following contents:

``` terraform	
terraform {
  required_version = "> 0.14"
  required_providers {
    azurerm = {
      version = "= 2.57.0"
    }
    kubernetes = {
      version = "= 2.1.0"
    }
    helm = {
      version = "= 2.1.2"
    }
  }
}

provider "azurerm" {
  features {}
}

# Configuring the kubernetes provider
# AKS resource name is aks: azurerm_kubernetes_cluster.aks
provider "kubernetes" {
  host                   = azurerm_kubernetes_cluster.aks.kube_config.0.host
  client_certificate     = base64decode(azurerm_kubernetes_cluster.aks.kube_config.0.client_certificate)
  client_key             = base64decode(azurerm_kubernetes_cluster.aks.kube_config.0.client_key)
  cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.aks.kube_config.0.cluster_ca_certificate)
}

# Configuring the helm provider
# AKS resource name is aks: azurerm_kubernetes_cluster.aks
provider "helm" {
  kubernetes {
    host                   = azurerm_kubernetes_cluster.aks.kube_config.0.host
    client_certificate     = base64decode(azurerm_kubernetes_cluster.aks.kube_config.0.client_certificate)
    client_key             = base64decode(azurerm_kubernetes_cluster.aks.kube_config.0.client_key)
    cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.aks.kube_config.0.cluster_ca_certificate)
  }
}
```  

### Create main.tf with the following contents:

``` terraform
# Create Resource Group
resource "azurerm_resource_group" "rg" {
  name     = var.resource_group
  location = var.location
}

# Create VNET for AKS
resource "azurerm_virtual_network" "vnet" {
  name                = "rook-network"
  address_space       = ["10.0.0.0/8"]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

# Create the Subnet for AKS.
resource "azurerm_subnet" "aks" {
  name                 = "aks"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.240.0.0/16"]
}

# Create the AKS cluster.
# Cause this is a test node_count is set to 1 
resource "azurerm_kubernetes_cluster" "aks" {
  name                = var.aks_name
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = var.aks_name
  kubernetes_version  = "1.21.2"

  default_node_pool {
    name               = "default"
    node_count         = 3
    vm_size            = "Standard_D2s_v3"
    os_disk_size_gb    = 30
    os_disk_type       = "Ephemeral"
    vnet_subnet_id     = azurerm_subnet.aks.id
    availability_zones = ["1", "2", "3"]
  }

  # Using Managed Identity
  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin = "azure"
    network_policy = "calico"
  }

  role_based_access_control {
    enabled = true
  }

  addon_profile {
    kube_dashboard {
      enabled = false
    }
  }
}

resource "azurerm_kubernetes_cluster_node_pool" "npceph" {
  name                  = "npceph"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.aks.id
  vm_size               = "Standard_DS2_v2"
  node_count            = 3
  node_taints           = ["storage-node=true:NoSchedule"]
  availability_zones    = ["1", "2", "3"]
  vnet_subnet_id        = azurerm_subnet.aks.id
}

data "azurerm_resource_group" "node_resource_group" {
  name = azurerm_kubernetes_cluster.aks.node_resource_group
}

resource "azurerm_role_assignment" "kubelet_contributor" {
  scope                = data.azurerm_resource_group.node_resource_group.id
  role_definition_name = "Contributor" #"Virtual Machine Contributor"?
  principal_id         = azurerm_kubernetes_cluster.aks.kubelet_identity[0].object_id
}

resource "azurerm_role_assignment" "identity_network_contributor" {
  scope                = azurerm_virtual_network.vnet.id
  role_definition_name = "Network Contributor"
  principal_id         = azurerm_kubernetes_cluster.aks.identity[0].principal_id
}
```

Please note the following:

* The AKS cluster is availability zone aware.
* A second node pool (npceph) is created for the Ceph storage. This pool is also availability zone aware.
* The node pool (npceph) is configured to use the `storage-node` taint.

### Create rook-ceph-operator-values.yaml with the following contents:

``` yaml
crds:
  enabled: true
csi:
  provisionerTolerations:
    - effect: NoSchedule
      key: storage-node
      operator: Exists
  pluginTolerations:
    - effect: NoSchedule
      key: storage-node
      operator: Exists
agent:
  # AKS: https://rook.github.io/docs/rook/v1.7/flexvolume.html#azure-aks
  flexVolumeDirPath: "/etc/kubernetes/volumeplugins"
```

This is the helm configuration for the rook-ceph-operator. As you can see both the provisioner and the plugin tolerations are using the `storage-node` taint.

### Create rook-ceph-cluster-values.yaml with the following contents:

``` yaml
operatorNamespace: rook-ceph
toolbox:
  enabled: true
cephBlockPools: []
cephObjectStores: []
cephClusterSpec:
  mon:
    volumeClaimTemplate:
      spec:
        storageClassName: managed-premium
        resources:
          requests:
            storage: 10Gi
  storage:
    storageClassDeviceSets:
      - name: set1
        # The number of OSDs to create from this device set
        count: 3
        # IMPORTANT: If volumes specified by the storageClassName are not portable across nodes
        # this needs to be set to false. For example, if using the local storage provisioner
        # this should be false.
        portable: false
        # Since the OSDs could end up on any node, an effort needs to be made to spread the OSDs
        # across nodes as much as possible. Unfortunately the pod anti-affinity breaks down
        # as soon as you have more than one OSD per node. The topology spread constraints will
        # give us an even spread on K8s 1.18 or newer.
        placement:
          topologySpreadConstraints:
            - maxSkew: 1
              topologyKey: kubernetes.io/hostname
              whenUnsatisfiable: ScheduleAnyway
              labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - rook-ceph-osd
          tolerations:
            - key: storage-node
              operator: Exists
        preparePlacement:
          tolerations:
            - key: storage-node
              operator: Exists
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
                - matchExpressions:
                    - key: agentpool
                      operator: In
                      values:
                        - npceph
          topologySpreadConstraints:
            - maxSkew: 1
              # IMPORTANT: If you don't have zone labels, change this to another key such as kubernetes.io/hostname
              topologyKey: topology.kubernetes.io/zone
              whenUnsatisfiable: DoNotSchedule
              labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - rook-ceph-osd-prepare
        resources:
          limits:
            cpu: "500m"
            memory: "4Gi"
          requests:
            cpu: "500m"
            memory: "2Gi"
        volumeClaimTemplates:
          - metadata:
              name: data
            spec:
              resources:
                requests:
                  storage: 100Gi
              storageClassName: managed-premium
              volumeMode: Block
              accessModes:
                - ReadWriteOnce
```

This is the configuration for the rook-ceph-cluster. Note that:

* The configuration deploys 3 OSDs.
* `storage-node` taint must be tolerated.
* topologySpreadConstraints are used to spread the OSDs across nodes.
* The toolbox is enabled.

### Create rook-ceph.tf with the following contents:

``` terraform
# Install rook-ceph using the hem chart
resource "helm_release" "rook-ceph" {
  name             = "rook-ceph"
  chart            = "rook-ceph"
  namespace        = "rook-ceph"
  version          = "1.7.3"
  repository       = "https://charts.rook.io/release/"
  create_namespace = true

  values = [
    "${file("./rook-ceph-operator-values.yaml")}"
  ]

  depends_on = [
    azurerm_kubernetes_cluster_node_pool.npceph
  ]
}

resource "helm_release" "rook-ceph-cluster" {
  name       = "rook-ceph-cluster"
  chart      = "rook-ceph-cluster"
  namespace  = "rook-ceph"
  version    = "1.7.3"
  repository = "https://charts.rook.io/release/"

  values = [
    "${file("./rook-ceph-cluster-values.yaml")}"
  ]

  depends_on = [
    azurerm_kubernetes_cluster_node_pool.npceph,
    helm_release.rook-ceph
  ]
}
```

### From the terraform folder run:

``` shell
terraform init
terraform apply
```

Once the cluster is deployed it will take a few minutes until the rook-ceph-cluster is ready.

Check that the OSDs are running:

``` shell	
az aks get-credentials --resource-group <resource group name> --name <aks name>
kubectl get pods -n rook-ceph
```

## Deploy a Test Application

### Create ceph-filesystem-pvc.yaml with the following contents:

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ceph-filesystem-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: ceph-filesystem
```

With this PVC you are asking for a 1Gi of storage from the ceph cluster.

### Create busybox-deployment.yaml with the following contents:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: busy
  name: busy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: busy
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: busy
    spec:
      containers:
        - image: busybox
          imagePullPolicy: Always
          name: busy-rook
          command:
            - sh
            - -c
            - test -f /ceph-file-store/important.file || echo "yada yada yada" >> /ceph-file-store/important.file && sleep 3600
          volumeMounts:
            - mountPath: "/ceph-file-store"
              name: ceph-volume
          resources: {}
      volumes:
        - name: ceph-volume
          persistentVolumeClaim:
            claimName: ceph-filesystem-pvc
            readOnly: false
```

The busybox deployment will mount the `ceph-filesystem-pvc` and will write a file to it.

### Deploy the test application using the following command:

``` shell
kubectl apply -f ceph-filesystem-pvc.yaml
kubectl apply -f busybox-deployment.yaml
```

### Check that everything is running as expected::

Check the pvc status:

``` shell	
kubectl get pvc
```

The output should look like this (Note the status is "Bound"):

``` shell	
NAME                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
ceph-filesystem-pvc   Bound    pvc-344e2517-b421-4a81-98e2-5bc6a991d93d   1Gi        RWX            ceph-filesystem   21h   
```

Check that the `important.file` exists:

``` shell	
kubectl exec $(kubectl get po -l app=busy -o jsonpath='{.items[0].metadata.name}') -it -- cat /ceph-file-store/important.file
```

you should get the contents of `important.file`:

``` shell
yada yada yada
```

## Performance Tests

You can run some performance tests using [kubestr](https://github.com/kastenhq/kubestr).

### Test the performance of the ceph storage:

Run:

``` shell	
.\kubestr.exe fio -z 20Gi -s ceph-filesystem
```

You should get some results:

``` shell	
PVC created kubestr-fio-pvc-6hvqr
Pod created kubestr-fio-pod-nmbmr
Running FIO test (default-fio) on StorageClass (ceph-filesystem) with a PVC of Size (20Gi)
Elapsed time- 2m35.1500188s
FIO test results:

FIO version - fio-3.20
Global options - ioengine=libaio verify=0 direct=1 gtod_reduce=1

JobName: read_iops
  blocksize=4K filesize=2G iodepth=64 rw=randread
read:
  IOPS=1583.366333 BW(KiB/s)=6350
  iops: min=1006 max=2280 avg=1599.766724
  bw(KiB/s): min=4024 max=9120 avg=6399.233398

JobName: write_iops
  blocksize=4K filesize=2G iodepth=64 rw=randwrite
write:
  IOPS=223.526337 BW(KiB/s)=910
  iops: min=124 max=305 avg=224.199997
  bw(KiB/s): min=496 max=1221 avg=897.133362

JobName: read_bw
  blocksize=128K filesize=2G iodepth=64 rw=randread
read:
  IOPS=1565.778198 BW(KiB/s)=200950
  iops: min=968 max=2214 avg=1583.266724
  bw(KiB/s): min=123904 max=283392 avg=202674.265625

JobName: write_bw
  blocksize=128k filesize=2G iodepth=64 rw=randwrite
write:
  IOPS=225.524933 BW(KiB/s)=29396
  iops: min=124 max=308 avg=227.033340
  bw(KiB/s): min=15872 max=39424 avg=29077.132812

Disk stats (read/write):
  -  OK
```

### Test the performance of the Azure Files Premium:

Run:

``` shell	
.\kubestr.exe fio -z 20Gi -s azurefile-csi-premium
```

You should get some results:

``` shell
PVC created kubestr-fio-pvc-mvf9v
Pod created kubestr-fio-pod-qntnw
Running FIO test (default-fio) on StorageClass (azurefile-csi-premium) with a PVC of Size (20Gi)
Elapsed time- 59.3141476s
FIO test results:

FIO version - fio-3.20
Global options - ioengine=libaio verify=0 direct=1 gtod_reduce=1

JobName: read_iops
  blocksize=4K filesize=2G iodepth=64 rw=randread
read:
  IOPS=557.804260 BW(KiB/s)=2247
  iops: min=260 max=1294 avg=644.807678
  bw(KiB/s): min=1040 max=5176 avg=2579.384521

JobName: write_iops
  blocksize=4K filesize=2G iodepth=64 rw=randwrite
write:
  IOPS=255.239807 BW(KiB/s)=1037
  iops: min=6 max=428 avg=292.037048
  bw(KiB/s): min=24 max=1712 avg=1168.333374

JobName: read_bw
  blocksize=128K filesize=2G iodepth=64 rw=randread
read:
  IOPS=537.072571 BW(KiB/s)=69278
  iops: min=260 max=1358 avg=622.115356
  bw(KiB/s): min=33280 max=173824 avg=79648.304688

JobName: write_bw
  blocksize=128k filesize=2G iodepth=64 rw=randwrite
write:
  IOPS=295.383789 BW(KiB/s)=38343
  iops: min=144 max=872 avg=340.846161
  bw(KiB/s): min=18432 max=111616 avg=43637.308594

Disk stats (read/write):
  -  OK
```

Note that those are the results of the FIO tests I ran, but I will not jump into any conclusions since I am not a performance expert.

## Simulate a node crash

Let's simulate a VM crash by deallocating one of the nodes:

``` shell
$resourceGroupName="aks-rook"
$aksName="aks-rook"
$resourceGroup=$(az aks show --resource-group $resourceGroupName --name $aksName --query "nodeResourceGroup" --output tsv)
$cephScaleSet=$(az vmss list --resource-group $resourceGroup --query "[].{name:name}[? contains(name,'npceph')] | [0].name" --output tsv)
az vmss deallocate --resource-group $resourceGroup --name $cephScaleSet --instance-ids 0
```

Check that you still have access to the important.file:

``` shell	
kubectl exec $(kubectl get po -l app=busy -o jsonpath='{.items[0].metadata.name}') -it -- cat /ceph-file-store/important.file
```

Hope it helps!!!

Please find the complete sample [here](https://github.com/cmendible/azure.samples/tree/main/aks_rook)

References:

* [Using Rook / Ceph with PVCs on Azure Kubernetes Service](https://partlycloudy.blog/2019/12/08/using-rook-ceph-with-pvcs-on-azure-kubernetes-service/)
* [Ceph Operator Helm Chart](https://github.com/rook/rook/blob/master/Documentation/helm-operator.md)
* [Ceph FlexVolume Configuration for AKS](https://rook.github.io/docs/rook/v1.7/flexvolume.html#azure-aks)
* [Ceph Storage Shared Filesystem](https://rook.github.io/docs/rook/v1.7/ceph-filesystem.html)
* [rook.io](https://rook.io/)
* [ceph.com](https://ceph.io/)