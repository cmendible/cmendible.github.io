---
author: Carlos Mendible
categories:
- kubernetes
- Azure
crosspost_to_medium: true
date: "2018-07-19T00:21:00Z"
description: 'At last: Network Policies in AKS with kube-router'
image: /assets/img/posts/kubernetes.png
published: true
# tags: kube-router network_policies
title: 'At last: Network Policies in AKS with kube-router'
---

For ages I've been waiting for a way to enforce netwok policies on AKS, so last weekend while I was googling around, I found this hidden gem posted by [Marcus Robinson](https://www.techdiction.com/bio/): [Enforcing Network Policies using kube-router on AKS](https://www.techdiction.com/2018/06/02/enforcing-network-policies-using-kube-router-on-aks/) and had to test the proposed solution.

**Prerequisites**:

* [Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/) deployed with [HTTP application routing](https://docs.microsoft.com/en-us/azure/aks/http-application-routing) enabled.
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) installed

## 1. Create a service exposed throuh your AKS DNS Zone
---

Let's start by deploying the following service to your Kubernetes cluster, by saving the following content to a file named **dni-function.yaml** and replacing **[YOUR_DNS__ZONE_NAME]** with the corresponding value of your service:

``` yml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: dni-function
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: dni-function
    spec:
      containers:
      - name: dni-function
        image: cmendibl3/dni:1.0.0
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: dni-function
spec:
  type: ClusterIP
  ports:
  - name:
    port: 80
  selector:
    app: dni-function
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: dni-function
  annotations:
    kubernetes.io/ingress.class: addon-http-application-routing
spec:
  rules:
  - host: dni-function.[YOUR_DNS__ZONE_NAME]
    http:
      paths:
      - backend:
          serviceName: dni-function
          servicePort: 80
        path: /
```

Now deploy it to Kubernetes:

``` powershell
kubectl apply -f ./dni-function.yaml
```

In a few seconds you'll have a working Web API (Validates Spanish National Identification Numbers).

Now test the service with the following command:

``` bash
curl -k http://dni-function.[YOUR_DNS__ZONE_NAME]/api/validate?dni=88410248L
```

which should return **true**.

## 2. Deploy kube-router
---

Thanks to [Marcus Robinson](https://www.techdiction.com/bio/) we can deploy [kube-router](https://github.com/cloudnativelabs/kube-router) to AKS:

``` bash
kubectl apply -f https://raw.githubusercontent.com/marrobi/kube-router/marrobi/aks-yaml/daemonset/kube-router-firewall-daemonset-aks.yaml
```

## 3. Deny all traffic to the service
---

Now let's try to deny all the traffic to the service, creating a **dni-function-deny-all.yaml** file with the following contents:

``` yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: dni-function-deny-all
spec:
  podSelector:
    matchLabels:
      app: dni-function
  ingress: []
```

Deploy the policy:

``` bash
kubectl apply -f ./dni-function-deny-all.yaml
```

Try calling the service again:

``` bash
curl -k http://dni-function.[YOUR_DNS__ZONE_NAME]/api/validate?dni=88410248L
```

This time you should get:

``` html
<html>
<head><title>502 Bad Gateway</title></head>
<body bgcolor="white">
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.13.12</center>
</body>
</html>
```

That's it! Your service is no longer available!!!

## 4. Allow only traffic from a specific pod

Create a **dni-function-allow-internal.yaml** file with the following contents:

``` yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: dni-function-allow-internal
spec:
  podSelector:
    matchLabels:
      app: dni-function
  ingress:
  - from:
      - podSelector:
          matchLabels:
            app: internal
```

Deploy the policy, which restricts the traffic to pods with the label: **app=internal**, and check that you still can't connect:

``` bash
kubectl apply -f ./dni-function-allow-internal.yaml
curl -k http://dni-function.[YOUR_DNS__ZONE_NAME]/api/validate?dni=88410248L
```

Now let's create a pod with the expected label and try calling the **dni-function** service from it:

``` bash
kubectl run internal-function-tester --rm -i -t --image=alpine --labels app=internal -- sh
wget -qO- --timeout=2 http://dni-function/api/validate?dni=88410248L
```

This time you should get **true** as the result!!!

To learn more about Network Policies check the [kubernetes-network-policy-recipes](https://github.com/ahmetb/kubernetes-network-policy-recipes) repo and feel free to download the code and files for this post [here](https://github.com/cmendible/kubernetes.samples/11.kube-router)