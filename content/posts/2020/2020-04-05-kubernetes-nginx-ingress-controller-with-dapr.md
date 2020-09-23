---
author: Carlos Mendible
categories:
- kubernetes
crosspost_to_medium: true
date: "2020-04-05T00:00:00Z"
description: 'Kubernetes NGINX ingress controller with Dapr'
images: ["/assets/img/posts/dapr.png"]
published: true
tags: ["dapr"]
title: 'Kubernetes NGINX ingress controller with Dapr'
---

In this post I'll show you how to expose your "Daprized" applications using and NGINX ingress controller.

## Prerequistes
* A working [kubernetes](https://kubernetes.io/) cluster with Dapr installed. If you need instructions please find them [here](https://github.com/dapr/docs/blob/master/getting-started/environment-setup.md#installing-dapr-on-a-kubernetes-cluster)

## Deploy an application to your Kubernetes cluster

I'll be using a simple Azure Function I created back in 2017 in the following post: **[Run a Precompiled .NET Core Azure Function in a Container](https://carlos.mendible.com/2017/12/28/run-a-precomplied-net-core-azure-function-in-a-container/)** which exposes a simple validation function.


Create a *function.yaml* file with the following contents:


```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: dni-function
  namespace: default
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: dni-function
      annotations:
        dapr.io/enabled: "true"
        dapr.io/id: "dni"
        dapr.io/port: "80"
    spec:
      containers:
        - name: dni-function
          image: cmendibl3/dni:1.0.0
          ports:
            - containerPort: 80
```

Note the *dapr.io* annotations used to instruct Dapr to inject the sidecar in your pod.

Now deploy the function into kubernetes:

```shell
kubectl apply -f ./function.yaml
```

## Deploy NGINX Ingress Controller with Dapr

We are going to "Daprize" the NGINX Ingress Controller so traffic flows as shown in the following picture [[1]](#references):

![Daprized NGINX Ingress Controller](/assets/img/posts/dapr-nginx-ingress.png)

In order to add the *dapr.io* annotations to the NGINX pod, create a *dapr-annotations.yaml* file with the following contents:

```yaml
controller:
  podAnnotations:
    dapr.io/enabled: "true"
    dapr.io/id: "nginx-ingress"
    dapr.io/port: "80"
```

and deploy the NGINX ingress controller:

```shell
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm install nginx stable/nginx-ingress -f .\dapr-annotations.yaml -n default
```

Since we´ll also be adding TLS termination to the mix, run the following commands to generate the certificate and deploy the corresponding secret into kubernetes: 

```shell
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=hydra/O=hydra"
kubectl create secret tls tls-secret --key tls.key --cert tls.cert
```

Now create the ingress rule (*ingress.yaml** file) with the following contents:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-rules
  namespace: default
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
    - hosts:
        - hydra
      secretName: tls-secret
  rules:
    - host: hydra
      http:
        paths:
          - path: /
            backend:
              serviceName: nginx-ingress-dapr
              servicePort: 80
```

Note that the rule is calling the *nginx-ingress-dapr* service which was created by Dapr when we deployed the Daprized version of the ingress controller. This means that all trafic with the *hydra* host will be sent to the Dapr sidecar of your NGINX controller pod. 

Deploy the ingress rule:

```shell
kubectl apply -f ./ingress.yaml
```

## Test the "Daprized" application

To test the application we'll need the public IP of the ingress service, so run the following command and copy the resulting IP address:

```shell
kubectl get service --selector=app=nginx-ingress,component=controller -o jsonpath='{.items[*].status.loadBalancer.ingress[0].ip}'
```

Now make a simple curl request, using the [Dapr invocation api specification](https://github.com/dapr/docs/blob/master/reference/api/service_invocation_api.md), to the application:

```shell
curl -k -H "Host: hydra" "https://<ingress ip>/v1.0/invoke/dni/method/api/validate?dni=54495436H"
```

If everything runs as expected you should get the following result:

```shell
true
```

Hope it helps!

Learn more about Dapr [here](https://github.com/dapr/docs/tree/master/) and Dapr service invocation [here](https://github.com/dapr/docs/tree/master/howto/invoke-and-discover-services)

## Learn More

* [How Distributed Application Runtime (Dapr) has grown since its announcement](https://cloudblogs.microsoft.com/opensource/2020/04/29/distributed-application-runtime-dapr-growth-community-update?WT.mc_id=AZ-MVP-5002618)
* [Announcing Azure Functions extension for Dapr
](https://cloudblogs.microsoft.com/opensource/2020/07/01/announcing-azure-functions-extension-for-dapr?WT.mc_id=AZ-MVP-5002618)

## References

[1] Image from the [Dapr Predentation deck](https://github.com/dapr/docs/blob/master/presentations/Dapr%20Presentation%20Deck.pptx)