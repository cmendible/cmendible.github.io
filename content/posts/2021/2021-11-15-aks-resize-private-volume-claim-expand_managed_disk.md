---
author: Carlos Mendible
categories:
- azure
- kubernetes
date: "2021-11-15T10:00:00Z"
description: 'AKS: Resize Private Volume Claim to expand a Managed Premium Disk'
images: ["/assets/img/posts/aks.png"]
draft: false
tags: ["aks", "persistent volume claim", "managed disk"]
title: 'AKS: Resize Private Volume Claim to expand a Managed Premium Disk'
---

If you deployed a private volume claim using the `managed-premium` storage class, then ran out of space and now you are searching how to expand the disk to a larger disk, this is how you can do it from scratch:

> `manage-premium` storage class is a premium storage class that allows volume expansion: `allowVolumeExpansion: true`.


## Create a private volume claim using a `managed-premium` storage class:

Create a `pvc.yaml` file with the following contents:

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 128Gi
  storageClassName: managed-premium
```

and deploy it to your cluster:

``` shell
kubectl apply -f pvc.yaml
```	

## Create a deployment that uses the PVC:

Create a `deployment.yaml` file with the following contents:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - mountPath: "/mnt/azure"
          name: volume
      volumes:
        - name: volume
          persistentVolumeClaim:
            claimName: nginx-pvc
```

and deploy it to your cluster:

``` shell
kubectl apply -f deployment.yaml
```

## Check the PVC status:

To check the status of the PVC, run the following command:

``` shell
kubectl get pvc -w
```

Create a test file by running the following command:

``` shell
kubectl exec -it $(kubectl get po -l "app=nginx" -o name) -- sh -c "echo 'Very important file' > /mnt/azure/test.file"
```

## Resize the PVC and expand the disk:

Scale the deployment to 0 replicas:

``` shell
kubectl scale --replicas=0 deployment nginx
```

Check the status of the attached disk:

``` shell
$resourceGroupName="<your aks resource group>"
$aksName="<your aks name>"
$resourceGroup=$(az aks show --resource-group $resourceGroupName --name $aksName --query "nodeResourceGroup" --output tsv)
az disk list --resource-group $resourceGroup --query "[[0].diskState, [0].diskSizeGb]"
```

You should get the following result:

``` shell
[
  "Unattached",
  256
]
```

**Note**: I only had one disk attached to the AKS cluster, so I am using the first disk.

Resize the PVC:

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 256Gi
  storageClassName: managed-premium
  volumeName: nginx-pv
```

and deploy it to your cluster:

``` shell
kubectl apply -f pvc.yaml
```

Scale the deployment back to 1 replicas:

``` shell
kubectl scale --replicas=1 deployment nginx
```

and check the po status:

``` shell
kubectl get po -w
```

## Check the size of the disk:

Check again the status and size of the attached disk:

``` shell
$resourceGroupName="<your aks resource group>"
$aksName="<your aks name>"
$resourceGroup=$(az aks show --resource-group $resourceGroupName --name $aksName --query "nodeResourceGroup" --output tsv)
az disk list --resource-group $resourceGroup --query "[[0].diskState, [0].diskSizeGb]"
```

You should get the following result:

``` shell
[
  "Attached",
  256
]
```

and finally check the contents of the test file:

``` shell
kubectl exec $(kubectl get po -l "app=nginx" -o name) -- sh -c "cat /mnt/azure/test.file"
```

Hope it helps!!!
