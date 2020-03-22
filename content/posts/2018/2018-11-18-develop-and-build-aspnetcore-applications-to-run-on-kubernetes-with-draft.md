---
author: Carlos Mendible
categories:
- dotnetcore
- kubernetes
- azure
crosspost_to_medium: true
date: "2018-11-18T10:00:00Z"
description: Develop and build ASP.NET Core applications to run on Kubernetes with
  Draft
image: /assets/img/posts/draft-logo.png
published: true
tags: ["aks", "acr", "draft"]
title: Develop and build ASP.NET Core applications to run on Kubernetes with Draft
url: /2018/11/18/develop-and-build-aspnetcore-applications-to-run-on-kubernetes-with-draft/
---

You start developing an ASP.NET Core application to run it in Kubernetes and suddenly you find yourself creating a docker file, building an image, pushing the image to a registry, creating both a deployment and a service definition for Kubernetes and you wonder if there is a tool out there to help you streamline the whole process.

Meet [Draft](https://github.com/Azure/draft) a tool that will not only help you create the artifacts needed to build and run applications in Kubernetes but to also build the container image and deploy it.

## Prerequisites:

1. A Kubernetes cluster and the [kubectl](https://kubernetes.io/docs/reference/kubectl/kubectl/) CLI tool
2. Installation of [Helm](https://github.com/Kubernetes/helm) on your Kubernetes cluster
3. A Container Registry

## Install and Configure Draft

To install Draft I run the following command:

``` shell
cinst -y draft
```

Then configure it:

``` shell
draft config set registry <registry full name>
```

Be sure to log on to the Container Registry:

``` shell
docker login <registry full name>
```

or if working with Azure Container Registry:

``` shell
az acr login --name <acrName>
```

## Create and prepare the application

Creat an ASP.NET Core MVC application and create the artifacts with Draft:

``` shell
dotnet new mvc
draft create --pack=csharp -a mvcdraft
```

Note that I've specified two parameters:

* **pack** to tell draft that we are using C# as the language.
* **a** to specify the name of the Helm release we will run on Kubernetes

## Deploy and test your application

Run the following command to build the container image and then deploy the application to Kubernetes:

``` shell
draft up
```

The first time you run the command it will take a while. The output should look similar to this:

``` shell
Draft Up Started: 'netcoredraft': 01CWKAZ79WR6W66PHHR2AFRSGC
netcoredraft: Building Docker Image: SUCCESS ⚓  (1.0008s)
netcoredraft: Pushing Docker Image: SUCCESS ⚓  (3.2891s)
netcoredraft: Releasing Application: SUCCESS ⚓  (2.7629s)
Inspect the logs with `draft logs 01CWKAZ79WR6W66PHHR2AFRSGC`
```

To test the application run:

``` shell
draft connect
```

The output should look similar to:

``` shell
Connect to netcoredraft:80 on localhost:50998
[netcoredraft]:       No XML encryptor configured. Key {ea9dac08-e7af-468c-9e00-cbe38ab40fa8} may be persisted to storage in unencrypted form.
[netcoredraft]: Hosting environment: Production
[netcoredraft]: Content root path: /app
[netcoredraft]: Now listening on: http://[::]:80
[netcoredraft]: Application started. Press Ctrl+C to shut down.
```

Browse to the url shown in the output (i.e. http://localhost:50998) and you should be good to go!!!

## Cleaning up

Once you finish your tests you can cleanup with the following command:

``` shell
draft delete
```

Hope it helps!!!