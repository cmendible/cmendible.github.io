---
author: Carlos Mendible
categories:
- kubernetes
- azure
crosspost_to_medium: true
date: "2019-04-16T06:00:00Z"
description: 'AKS & Application Gateway: Expose more than one service in an ingress resource'
images: ["/assets/img/posts/aks.png"]
published: true
tags: ["applicationgateway", "waf", "ingress"]
title: 'AKS & Application Gateway: Expose more than one service in an ingress resource'
url: /2019/04/16/aks-application-gateway-expose-more-than-one-service-in-a-ingress-resource/
---

If you install the Azure Application Gateway Ingress Controller for your AKS clusters you may want to expose more than one service through the same Public IP just changing the url path. In order to make this work you must use the [backend-path-prefix](https://github.com/Azure/application-gateway-kubernetes-ingress/blob/master/docs/annotations.md#backend-path-prefix) annotation.

In the following sample I create an ingress with the following behavior:

1. Calls to **http://[Waf Pulic IP or DNS]/inventory** are sent to the **inventory** service
1. Calls to **http://[Waf Pulic IP or DNS]/orders** are sent to the **orders** service

``` yaml
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: backend-ingress
  annotations:
    # Service are serving content in this path.
    appgw.ingress.kubernetes.io/backend-path-prefix: "/"
    kubernetes.io/ingress.class: azure/application-gateway
spec:
  rules:
  - http:
      paths:
      - path: /inventory/*
        backend:
          serviceName: inventory
          servicePort: 80
      - path: /orders/*
        backend:
          serviceName: orders
          servicePort: 80
```

Note that the **appgw.ingress.kubernetes.io/backend-path-prefix** value is set to: **/** which means that the paths specified in the rules are rewritten to this value when sent to your services.

Hope it helps.
