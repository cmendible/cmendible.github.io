---
author: Carlos Mendible
categories:
- azure
crosspost_to_medium: true
date: "2017-12-01T15:13:00Z"
description: Deploy your first Service to Azure Container Services (AKS)
images: ["/assets/img/posts/aks.png"]
published: true
tags: ["Docker", "kubernetes"]
title: Deploy your first Service to Azure Container Services (AKS)
url: /2017/12/01/deploy-your-first-service-to-azure-container-services-aks/
---

In this post I'll show you how to **Deploy your first Service to [Azure Container Services (AKS)](https://docs.microsoft.com/en-us/azure/aks/)**.

**Prerequisites**:

* [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) installed and basic knowledge experience.
* [Docker](https://www.docker.com) installed and basic knowledge.
* [Azure Subscription](https://azure.microsoft.com/en-us/pricing/purchase-options/)
* [Kubernetes](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/) experience.

## 1. Create a resource group:

Firt create a Resource Group. Be aware that at the time of writing AKS is not available in all Azure regions. 

``` powershell
az group create -l westeurope -n aks
```

## 2. Create an Azure Container Registry:

Probably you'll want a private registry to upload your docker images, so let's create an [Azure Container Registry](https://azure.microsoft.com/en-us/services/container-registry/) instance.

``` powershell
az acr create -g aks -n myregistry --sku Basic --admin-enabled
```

## 3. Login to the Azure Container Registry:

You can login to the Container Registry using the following Azure CLI command:

``` powershell
az acr login -n myregistry
```

Or you can get the Key and login with docker:

``` powershell
az acr credential show -n myregistry
docker login myregistry.azurecr.io -u myregistry -p [PASSWORD FROM PREVIOUS COMMAND]
```

## 4. Push an image to the Azure Container Registry:

In this step we are going to pull an image from docker hub, and then upload it to the Container Registry created in step 2. Feel free to use your own docker image with a working web application.

``` powershell
docker pull yaros1av/hello-core
docker tag yaros1av/hello-core myregistry.azurecr.io/hello-core:1.0
docker push myregistry.azurecr.io/hello-core:1.0
```

## 5. Create AKS cluster

Now create the AKS cluster:

``` powershell
az aks create -g aks --name myAKSCluster --generate-ssh-keys
```

## 6. Install [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/)

To manage the cluster you'll need to install [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/) so run the following command:

``` powershell
az aks install-cli
```

## 7. Get cluster credentials

Get the cluster credentials and check the connectivity so you can start working with the AKS cluster.

``` powershell
az aks get-credentials -g aks -n myAKSCluster
kubectl get nodes
```

## 8. Create a Secret to hold the registry credentials.

You have to add the Azure Container Registry credentials to your AKS service in order to be able to pull the images. This is acomplished by creating a secret:

``` powershell
kubectl create secret docker-registry [SECRET NAME] --docker-server=myregistry.azurecr.io --docker-username=myregistry --docker-password=[THE REGISTRY PASWORD FROM STEP 3] --docker-email=[EMAIL ADDRESS]
```

Check for the secret:

``` powershell
kubectl describe secret
```

## 9. Create a kubernetes [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) definition.

Create a file **[your deployment].yml** with the following contents (replace the secret name):

``` yml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: my-api
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: my-api
    spec:
      containers:
      - name: my-api
        image: myregistry.azurecr.io/hello-core:1.0
        ports:
        - containerPort: 80
      imagePullSecrets:
      - name: [SECRET NAME]
---
apiVersion: v1
kind: Service
metadata:
  name: my-api
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: my-api
```

As you can see the file references the image pushed to the Container Service with the secret created in previous steps.

## 10. Deploy your service.

Run the following comnmand to deploy your service to AKS:

``` powershell
kubectl create -f [your deployment].yml
```

## 11. Get the puplic IP for your service

To try your service you'll need to get the public IP (It can take a while).

``` powershell
kubectl.exe get service/my-api -w
```

Once you get the IP go ahead and test the service.

## 12. Update your deployment to different version of the image

If you need to update your service to use another version of the dokcer image run the following command:

``` powershell
kubectl set image deployment/my-api [image name]=myregistry.azurecr.io/[image name]:[new version]
```

## 13. Add an autoscale rule

To add an autoscale rule to your service, run the following command:

``` powershell
kubectl autoscale deployment my-api --min=2 --max=5 --cpu-percent=80
```

Hope it helps!