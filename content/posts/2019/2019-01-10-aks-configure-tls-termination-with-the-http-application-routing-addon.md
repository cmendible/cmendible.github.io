---
author: Carlos Mendible
categories:
- kubernetes
- azure
crosspost_to_medium: true
date: "2019-01-10T15:20:00Z"
description: 'AKS: Configure TLS termination with the http application routing addon'
images: ["/assets/img/posts/aks.png"]
published: true
tags: ["nginx", "ingress", "tls", "routing"]
title: 'AKS: Configure TLS termination with the http application routing addon'
---

When you install a AKS cluster you can configure it to deploy the [http application routing addon](https://docs.microsoft.com/en-us/azure/aks/http-application-routing) or you you can update an existing cluster to deploy it.

Either way you end up with an [NGINX Ingress Controller](https://github.com/kubernetes/ingress-nginx) running, in the **kube-system** namespace of your cluster, with the following properties:

* *ingress-class: addon-http-application-routing*
* *annotations-prefix: nginx.ingress.kubernetes.io*

Does this means that you can use this controller for TLS termination? The answer is yes! And you can also use rate limits, and whitelisting as described in my post [Secure your Kubernetes services with NGINX ingress controller, tls and more.](https://carlos.mendible.com/2018/03/20/secure-your-kubernetes-services-with-nginx-ingress-controller-tls-and-more/)

So to try it out, follow steps **2** and **5** of the previous post, but be sure to replace the contents of the **ingress_rules.yaml** file with the following yaml (Don't forget to replace the DNS Zone Name):

``` yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: dni-function
  namespace: default
  annotations:
    kubernetes.io/ingress.class: addon-http-application-routing
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - dni-function.<YOUR CLUSTERS DNS ZONE NAME>
    secretName: tls-secret
  rules:
  - host: dni-function.<YOUR CLUSTERS DNS ZONE NAME>
    http:
      paths:
      - path: /
        backend:
          serviceName: dni-function
          servicePort: 80
```

Note that the **kubernetes.io/ingress.class** value must be: **addon-http-application-routing**

Once you have tls working go ahead and try rate limits and whitelisting!

Hope it helps.

Please download all code and files [here](https://github.com/cmendible/kubernetes.samples/tree/master/15.http-application-routing-tls) and be sure to check the [online documentation](https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/nginx-configuration/annotations.md) to learn more about the annotations and other NGINX features.