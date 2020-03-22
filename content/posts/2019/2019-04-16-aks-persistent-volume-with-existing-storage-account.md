---
author: Carlos Mendible
categories:
- kubernetes
- azure
crosspost_to_medium: true
date: "2019-04-16T05:00:00Z"
description: 'AKS: Persistent Volume with existing Storage Account'
image: /assets/img/posts/kubernetes.png
published: true
tags: ["StorageAccount", "persistentvolume", "persistentvolumeclaim", "storageclass"]
title: 'AKS: Persistent Volume with existing Storage Account'
---

In order to deploy a Persistent Volume in your AKS cluster using an existing Storage Account you should take the following steps:

1. Create a **Storage Class** with a reference to the Storage Account.
1. Create a **Secret** with the credentials used to access the Storage Account.
1. Create a **Persistent Volume** with a reference to the Storage Class, the secret and the File Share.
1. Create a **Persistent Volume Claim** with a reference to the volume by name.

Use the following **yaml** as a template for the resources described above. Save the contents as **aks-existing-storage-account-pv.yaml**:

``` yaml
---
# Create a StorageClass object pointing to the existing Storage Account
# Remember: that the Storage account must be in the same Resource Group where the AKS cluster is deployed
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: azurefile
provisioner: kubernetes.io/azure-file
mountOptions:
  - dir_mode=0777
  - file_mode=0777
parameters:
  storageAccount: <storage account name>
  location: <storage account location>

---
# Create a Secret to hold the name and key of the Storage Account
# Remember: values are base64 encoded
apiVersion: v1
kind: Secret
metadata:
  name: azurefile-secret
type: Opaque
data:
  azurestorageaccountname: <base64 encoded storage account name>
  azurestorageaccountkey: <base64 encoded storage account key>

---
# Create a persistent volume, with the corresponding StorageClass and the reference to the Azure File secret.
# Remember: Create the share in the storage account otherwise the pods will fail with a "No such file or directory"
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  storageClassName: azurefile
  azureFile:
    secretName: azurefile-secret
    shareName: <Share Name (must already exist in the storage account)>
    readOnly: false
  mountOptions:
  - dir_mode=0777
  - file_mode=0777
  - uid=1000
  - gid=1000

---
# Create a PersistentVolumeClaim referencing the StorageClass and the volume
# Remember: this is a static scenario. The volume was created in the previous step.
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  accessModes:
    - ReadWriteOnce  
  resources:
    requests:
      storage: 5Gi
  storageClassName: azurefile
  volumeName: nginx-pv
```

Deploy to your cluster and verify that the Private Volume Claim status is **Bound**:

``` shell
kubectl apply -f aks-existing-storage-account-pv.yaml
kubectl get pvc
```

Result should show something like:

``` shell
NAME                              STATUS    VOLUME                      CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nginx-pvc                         Bound     nginx-pv                    5Gi        RWO            azurefile      ...
```

That's it! now you can mount a volume in a container with a reference to the Private Volume Claim as in the following deployment:

``` yaml
---
# Deploy an nginx mounting a volume and referencing the persisten volume claim
# Remember: using pvc decouples your deployment from the volume implementations
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
spec:  
  template:
    metadata:
      labels:
        app: nginx-storage
    spec:
      containers:
      - name: nginx-pod
        image: nginx:1.15.5
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
        volumeMounts:
        - mountPath: "/mnt/azure"
          name: volume
      volumes:
        - name: volume
          persistentVolumeClaim:
            claimName: nginx-pvc
```

Hope it helps.

Please download all code and files [here](https://github.com/cmendible/kubernetes.samples/tree/master/16.aks-pv-with-existing-storage-account).