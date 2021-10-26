---
author: Carlos Mendible
categories:
- kubernetes
- azure
- devops
date: "2020-02-09T00:00:00Z"
description: 'MongoDB Enterprise Operator: Deploying MongoDB in AKS'
images: ["/assets/img/posts/aks.png"]
published: true
tags: ["aks", "mongodb"]
title: 'MongoDB Enterprise Operator: Deploying MongoDB in AKS'
url: /2020/02/09/mongodb-enterprise-operator-deploying-mongodb-in-aks/
---

a couple of weeks ago I was trying to deploy MongoDB in AKS using the [MongoDB Enterprise Operator](https://docs.mongodb.com/kubernetes-operator/master/tutorial/install-k8s-operator/) and had trouble finding a simple tutorial to make the thing work. This post intends to fill that gap with a straight to the point approach.

## Prerequisites

Be sure to deploy AKS with a set of nodes with at least 8GB of RAM. I used **Standard_D3_v2**

## First clone the MongoDB Enterprise Kubernetes repo 

``` shell
git clone https://github.com/mongodb/mongodb-enterprise-kubernetes.git
cd mongodb-enterprise-kubernetes
```

---

## Create the MongoDB namespace inside your cluster

``` shell
kubectl create namespace mongodb
```

---

## Deploy the Custom Resource Definitions

``` shell
kubectl apply -f crds.yaml
```

---

## Deploy the MongoDB Enterprise Operator

``` shell
kubectl apply -f mongodb-enterprise.yaml
```

---

## Create a secret for the Ops Manager

As part of the installation you'll have to deploy the Ops Manager to the cluster. Be sure to set a complex password otherwise the Ops Manager wont start.

``` shell
kubectl create secret generic ops-manager-admin-secret `
  --from-literal=Username="<Username>" `
  --from-literal=Password="<Password>" `
  --from-literal=FirstName="<Name>" `
  --from-literal=LastName="<Last Name>" `
  -n mongodb
```

---

## Deploy the Ops Manager

Create **ops-manager.yaml** with the following contents (Note that we are using 1 replica):

``` yaml
---
apiVersion: mongodb.com/v1
kind: MongoDBOpsManager
metadata:
  name: ops-manager
spec:
  # the number of Ops Manager instances to run. Set to value bigger
  # than 1 to get high availability and upgrades without downtime
  replicas: 1

  # the version of Ops Manager distro to use
  version: 4.2.4

  # the name of the secret containing admin user credentials.
  # Either remove the secret or change the password using Ops Manager UI after the Ops Manager
  # resource is created!
  adminCredentials: ops-manager-admin-secret

  # the Ops Manager configuration. All the values must be of type string
  configuration:
    mms.fromEmailAddr: "admin@thecompany.com"

  # the application database backing Ops Manager. Replica Set is the only supported type
  # Application database has the SCRAM-SHA authentication mode always enabled
  applicationDatabase:
    members: 3
    version: 4.2.0
    persistent: true
    podSpec:
      cpu: "0.25"
```

Deploy the Ops Manager:

``` shell
kubectl apply -f .\ops-manager.yaml -n mongodb
```

---

## Wait until the Ops Manager is ready

``` shell
kubectl get om -n mongodb -o yaml -w
```

---

## Port Forward so you can access Ops Manager from your PC

``` shell
kubectl port-forward pods/ops-manager-0 8080:8080 -n mongodb
```

---

## Setup de Ops Manager

1. Login into the Ops Manager (http://localhost:8080) using the same user and password you deployed as a secret.

2. [Create an Organization](http://docs.opsmanager.mongodb.com/current/tutorial/manage-organizations/). Copy the **Organnization Id** so you can use it later.

3. [Create Public & Private Key for the Organization](https://docs.opsmanager.mongodb.com/rapid/tutorial/configure-public-api-access/#configure-public-api-access). Copy both keys so you can use them later.

4. [White List the Operator IPs](https://docs.opsmanager.mongodb.com/rapid/tutorial/configure-public-api-access/#create-org-app-api-key). To get the IPs run:

``` shell
kubectl get po --selector=app=mongodb-enterprise-operator -n mongodb -o jsonpath='{.items[*].status.podIP}'
```

---

## Create a secret that will be used by the operator to connect with the ops manager

Use the Public and Private keys copied in previous steps to run tyhe following command:

``` shell
kubectl -n mongodb `
  create secret generic my-credentials `
  --from-literal="user=<private key>" `
  --from-literal="publicApiKey=<public key>"
```

---

## Create a project

Create **project.yaml** with the following contents and be sure to replace the **Organization Id** you copied in previous steps:

``` yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-test
data:
  projectName: mongodb-test
  baseUrl: http://ops-manager-svc.mongodb.svc.cluster.local:8080

  orgId: <Organization Id>
```

Deploy the project ConfigMap definition:

``` shell
kubectl apply .\project.yaml -n mongodb
```

---

## Create a standalone MongoDB instance

Create **standalone.yaml** with the following contents:

``` yaml
apiVersion: mongodb.com/v1
kind: MongoDB
metadata:
  name: my-standalone
spec:
  version: 4.2.1
  type: Standalone
  opsManager:
    configMapRef:
      name: mongodb-test
  credentials: my-credentials

  # This flag allows the creation of pods without persistent volumes. This is for
  # testing only, and must not be used in production. 'false' will disable
  # Persistent Volume Claims. The default is 'true'
  persistent: false
```

Deploy the standalone MongoDB:

``` shell
kubectl apply -f .\standalone.yaml -n mongodb
```

---

Enjoy! You have MongoDB up and running!

To learn more about the MongoDB Enterprise Operator please check [here](https://docs.mongodb.com/kubernetes-operator/master/tutorial/install-k8s-operator/).