---
author: Carlos Mendible
categories:
- kubernetes
- azure
crosspost_to_medium: true
date: "2018-03-20T15:13:00Z"
description: Secure your Kubernetes services with NGINX ingress controller, tls and
  more.
image: /assets/img/posts/kubernetes.png
published: true
tags: ["nginx", "ingress", "whitelist", "tls", "ratelimit"]
title: Secure your Kubernetes services with NGINX ingress controller, tls and more.
url: /2018/03/20/secure-your-kubernetes-services-with-nginx-ingress-controller-tls-and-more/
---

**Disclaimer:** samples provided in this post were tested both in [Azure Container Services (AKS)](https://docs.microsoft.com/en-us/azure/aks/) and [Kubernetes provided by Docker for Windows](https://blog.docker.com/2018/01/docker-windows-desktop-now-kubernetes/).

In previous posts I showed you how to [Run a Precompiled .NET Core Azure Function in a Container]({{ site.baseurl }}{% post_url 2017-12-28-run-a-precomplied-net-core-azure-function-in-a-container %}) and how to [Deploy your first Service to Azure Container Services (AKS)]({{ site.baseurl }}{% post_url 2017-12-01-deploy-your-first-service-to-azure-container-services-aks %}).

By now you should be able to run your own services in Kubernetes and starting to wonder about how can you give answers to questions such as:

* Is there an easy way add TLS support to my services?
* Can I add whitelisting and other nice features such as rate limits ("throtling")?

Through this post I'll walk you through a series of samples to show you how you can answer those questions by deploying a [NGINX Ingress Controller](https://github.com/kubernetes/ingress-nginx) to your Kubernetes cluster.

**Prerequisites**:

* [Kubernetes](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/) deployed and working experience.
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) installed
* [helm](https://github.com/kubernetes/helm)

## 1. Expose a service with public IP
---

Let's start by deploying the following service to your Kubernetes cluster, by saving the following content yaml content to a file named **dni-function.yaml**:

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
  type: LoadBalancer
  ports:
  - name:
    port: 9000
    targetPort: 80
  selector:
    app: dni-function
```

Now deploy it to Kubernetes:

``` powershell
kubectl apply -f ./dni-function.yaml
```

In a few seconds you'll have a working Web API (Validates Spanish National Identification Numbers) with the service type set to **LoadBalancer** which means that the solution is exposed to the world using a public IP.

If you are running on localhost, you can try it executing:

``` bash
curl -k http://localhost:9000/api/validate?dni=88410248L
```

which should return **true**.

## 2. Make the service private (No public IP)
---

Now in order to secure the API let's make the service private changing it's type to **ClusterIP**. So update the contents of **dni-function.yaml** as follows:

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
```

and redeploy:

``` powershell
kubectl delete -f ./dni-function.yaml
kubectl apply -f ./dni-function.yaml
```

Now you don't have a public endpoint and therefore any attempt to query the service will result in a Service Unavailable response.

## 3. Deploy the [NGINX Ingress Controller](https://github.com/kubernetes/ingress-nginx)
---

It's time to deploy the [NGINX Ingress Controller](https://github.com/kubernetes/ingress-nginx): a daemon, deployed as a Kubernetes Pod which provides a simple yet effective way to configure features such as TLS, Whitelisting, rate limits, etc...

Let's use Helm to deploy the ingress controller:

``` powershell
helm install stable/nginx-ingress --name my-nginx `
    --set controller.ingressClass=nginx `
    --set controller.service.externalTrafficPolicy=Local `
    --set controller.service.loadBalancerIP=127.0.0.1
```

If you test again with:

``` bash
curl -k http://localhost/api/validate?dni=88410248L
```

and because there is no rule specified to route the traffic from the ingress endpoint to the private service, you'll get a response coming from a **default backend!**

## 4. Deploy the first Ingress Rule
---

Create a file named **ingress_rules.yaml** with the following contents and deploy:

``` yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-rules
  namespace: default
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: localhost
    http:
      paths:
      - path: /
        backend:
          serviceName: dni-function
          servicePort: 80
```

``` bash
kubectl appply -f ingress_rules.yaml
```

If everything is ok and you query the service again you should get a response from the Web API, so we are back to square one, but with a great difference: all traffic goes through the [NGINX Ingress Controller](https://github.com/kubernetes/ingress-nginx) and that fact will let us add new features to the solution.

Please note that with this rules you can use different paths to expose diferent services.

## 5. TLS
---

To add TLS to the service you'll first need a certificate which you can generate with [openssl](https://github.com/kubernetes/ingress-nginx/blob/master/docs/examples/PREREQUISITES.md#tls-certificates). In order to speed things up save the following contents to a file named **tls-secret.yaml**, which provides a certificate for localhost (just for testing) and will add a [secret](https://kubernetes.io/docs/concepts/configuration/secret/) to your Kubernetes cluster:

``` yml
apiVersion: v1
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMrekNDQWVPZ0F3SUJBZ0lKQVBBVlNoS1d5NWw1TUEwR0NTcUdTSWIzRFFFQkN3VUFNQlF4RWpBUUJnTlYKQkFNTUNXeHZZMkZzYUc5emREQWVGdzB4T0RBeU1qVXhOelUzTURSYUZ3MHhPVEF5TWpVeE56VTNNRFJhTUJReApFakFRQmdOVkJBTU1DV3h2WTJGc2FHOXpkRENDQVNJd0RRWUpLb1pJaHZjTkFRRUJCUUFEZ2dFUEFEQ0NBUW9DCmdnRUJBT012c25IOEhmT2dSSnE5UDRLdGRudnNkWEtaa0tYMHpNclpNdURHWnppeXFucEMwVzdwRVZOV0lmK2kKcFNpcGU2RWNBeDlWakNmUWxZbHJuNnZBLzFLUTlLRW84SURicUF3ZTloVGFLK2dRV21IblJGaFhFcCs5ZmxqbQptcVRQS3JHNEgydnlZTWQwYnpacUlJQUZHMTV3NzZLODZNRzhxVUR0V0NRNWxvWFNBZUtUVzcwSVNMSXVNZ016CklpV2lsVjB4Ly9MS2RHZDJKRi9wZVVBUGhqT05SNUJ2cHArUXlhQ3gvTDNFaGFuWE56amZXUi8zZ084SUZwN0IKcmhpcWwzWWNCZHJES2xTaE8raTFDY3FnbGZUdEdkaXVENWZHYXB3QUQ5eVNwV081bTFSR1NTaHZmTktHR05XVApOeDczemQ4b0FMckFHbHR6K2I4VjlSaTMrNThDQXdFQUFhTlFNRTR3SFFZRFZSME9CQllFRlB2S1BFT0U2VUFnCnVLTFA5QjRDczkzc2d3SnRNQjhHQTFVZEl3UVlNQmFBRlB2S1BFT0U2VUFndUtMUDlCNENzOTNzZ3dKdE1Bd0cKQTFVZEV3UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFDYVM0eTFZMnVjdzRsOEJMVGJ0NW9QVApnQXpKSW5iYTVET3NWZElEcmZEa0FOOUJhcVh2Y011dzlmN0xYc0dlenNiMmdCSm1GbllpcUxLcTBYdmVBU2x2ClJoS0R5OVJhTThpeTV0eVdVZEJXSzJFZWtvUEJHOFhMVHhScnorbEpoQmpMNXhTS3dSMDN3MW45OEVQbWRFOUkKTS90OWdHKzdZYnZyaVZhdTFINURiZWtTZld0L2pIZG9UU2hwNEJFTVNxelJSaHpqbmNUamZoR0J2KzR2cGFCRAo1aHZ0WHFsekZHODdRR05LQWdHK01hdFFLY1JCTFppNndkRDk4djdMV2NROGhydWxGT2lxeXNmNDh3bzZodEhyCm14S3BnVWpHbE52dVdKZmhNWTZCdlkyeFlveHB2R0d3Z1hnNWsxQUFldzNjK0RRTWpUUjU3dnVTZ2tnTGZVaz0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
  tls.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUV2Z0lCQURBTkJna3Foa2lHOXcwQkFRRUZBQVNDQktnd2dnU2tBZ0VBQW9JQkFRRGpMN0p4L0Izem9FU2EKdlQrQ3JYWjc3SFZ5bVpDbDlNeksyVExneG1jNHNxcDZRdEZ1NlJGVFZpSC9vcVVvcVh1aEhBTWZWWXduMEpXSgphNStyd1A5U2tQU2hLUENBMjZnTUh2WVUyaXZvRUZwaDUwUllWeEtmdlg1WTVwcWt6eXF4dUI5cjhtREhkRzgyCmFpQ0FCUnRlY08raXZPakJ2S2xBN1Zna09aYUYwZ0hpazF1OUNFaXlMaklETXlJbG9wVmRNZi95eW5SbmRpUmYKNlhsQUQ0WXpqVWVRYjZhZmtNbWdzZnk5eElXcDF6YzQzMWtmOTREdkNCYWV3YTRZcXBkMkhBWGF3eXBVb1R2bwp0UW5Lb0pYMDdSbllyZytYeG1xY0FBL2NrcVZqdVp0VVJra29iM3pTaGhqVmt6Y2U5ODNmS0FDNndCcGJjL20vCkZmVVl0L3VmQWdNQkFBRUNnZ0VCQUpqbEhza0xqZlRLSmFHbVA3bm9sOWJxMmxnWDlYdGE5d0NGa0hJcDFJb1oKNUJXSUpuN29LQnJYMnVXNlJrREpYMFNjSDVYVTh4QlFsbkwzbFd2MzVWMWg1T0VaTmxMaWdZUTJ5aEphaWpZUgoyMklNVExqUFVOOWtua1dpWE8wUjUzL1hsSDRIanc1czAvUGhGS0pUelltUHBCYjM0QVdTdkszUGpnUkRKWVJFCi9ybEF0SzZSaHFKNThBS0dPOGp3OEd5WEJhTzJRVkVJSTVVdUM1QTBhTGMvUjJxS1ZpdzJ6bGJ6TVRNUVpKc2QKVjRubklYS29oL1R6WWZjdk90OVZKOTRqWlRmSGtRdHBHbzgwbFVhNGtFMXY2b2gyZG5GcThtZ3M5a3ZDV2ZLSwpTTlVhNkU2UGlMenFzR01VRE85enVHRmpUQm9xMDdBamNSVHMvS1NaOEdFQ2dZRUEvUGlkTUtGQnRaMHp6TnpOClB0Q1FrMzZkZ3VMZ2xhSVl5Qk5JSXl5SG84WW5wS3dONWJ1ZjM2ZktIWmZMeUt3SnNFMkhiNWQ0aXI1RzNFaVAKYzBhYjJFQnpQMTdyaUVVb2htVkJZTUNjY1ZlMi9qWnduZ2ducjBpcFEvc3ZqSFdrYlBTcFozam9ySjV4Z1owYQpXM21yN0U1Mi85THQxTGFYeXdRMkIydDAvR2tDZ1lFQTVlZ01yVzIzMlIyS3NrQVRDWDV3UDNwYk9SUkJkaEU5Cmh0a2JDTXc0ZDFHU1hBOElWSEVKRkZVdkFjODhiY0ZlckxXQzZkd0V1N2pTRnUyRUwwcEdKWmFTQ2NoTllrUkIKQ1F1MFhOdDBzUzRoRzhoUFRiejZROVZ6RzBjVm91RUlsdlVEc0dCU3pOV2FYck5lSDY5MktPMWM5akd4RzVITgozTC9VdzM2STFzY0NnWUJYMUhHdkNxM253bmJUciszSzIxcjIrc1R4UnBnM0c1cURETDdGQjVib2M4b2IwR2phCjFIUERrVndKUGtUUW5YcVhyYk5TT1VMdTJQVjlVZXdNVi8yUDdZQ1dCZnk4eVZZeW8wRTV1R1lZckIycTBYZjAKUmx5UTdTZG5wUFJ6VGYwU256ZVo1MDdSY0FsMHVQa0h2WXpGZE5DNExhSEpjc1B0QnI5RGdEbVQwUUtCZ0NDbQpPcDZxZFRCdEpKUTV5enBPN1d2bVdXd2F0MDBvRjUrOTF6d0JuSWM5VzFhZGYrWldBeDhURmREZytFang3QnNFCnorbWNLRVBzZEZGek81Rm5yOXlJcklhZEhuZzFEek5VcVRHQ3JPaTRqMVVkdGoxbzkvV0lLNGVWS2JwdTBNUjMKV1NYRUdCNGt1MzUxWkltRlpuZGJkaGMwYVYxcjhGdElGdFFJZFRCakFvR0JBSXV1dGFzQjFFT2dUZUVtTGJqYgpKdWVlZVRtSUQvWHpLMldpSE1ydG5HOHN0TWF0ai9VWjZUcTJMQnh4SkJqdVRFOHF5dUtYeG1KTmUxSnRkbUdKClU2bkZmWXo4OW1QSmlWK2dtSWU1L1VNNGJRbnB2SFNJdEtyNHFCREgvTkY0Z21xdGQ3MDlnTm5XRS9jdk1sQWkKQndYNWh2TzFWdndzZmRNeE0rSGIwVUxPCi0tLS0tRU5EIFBSSVZBVEUgS0VZLS0tLS0K
kind: Secret
metadata:
  name: tls-secret
  namespace: default
type: Opaque
```

Now modify the **ingress_rules.yaml** from previous steps to include the **tls** section as shown below:

``` yaml
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
    - localhost
    secretName: tls-secret
  rules:
  - host: localhost
    http:
      paths:
      - path: /
        backend:
          serviceName: dni-function
          servicePort: 80
```

Deploy both files:

``` bash
kubectl apply -f ./tls-secret.yaml
kubectl apply -f ./ingress_rules.yaml
```

In this new state, if you query the service you should get redirects to the TLS endpoint, so try:

``` bash
https://localhost/api/validate?dni=88410248L;
```

and yes! the service is responding and only accesible through the TLS endpoint!

## 6. Whitelisting
---

To restrict the service in a way that only a list of IPs can access it, modify the **ingress_rules.yaml** to add the **whitelist-source-range** annotation:

``` yml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-rules
  namespace: default
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/whitelist-source-range: '192.168.65.3/32'
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - localhost
    secretName: tls-secret
  rules:
  - host: localhost
    http:
      paths:
      - path: /
        backend:
          serviceName: dni-function
          servicePort: 80
```

and deploy:

``` bash
kubectl apply -f ./ingress_rules.yaml
```

Feel free to try different ranges and understand how you can block or enable access to your service.

## 7. Rate limits
---

Now it's time to protect the service applying some kind of throtling. Modify the **ingress_rules.yaml** to add the **limit-connections** and **limit-rps** annotations:

``` yml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-rules
  namespace: default
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/whitelist-source-range: '192.168.65.3/32'
    nginx.ingress.kubernetes.io/limit-connections: '10'
    nginx.ingress.kubernetes.io/limit-rps: '1'
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - localhost
    secretName: tls-secret
  rules:
  - host: localhost
    http:
      paths:
      - path: /
        backend:
          serviceName: dni-function
          servicePort: 80
```

and deploy:

``` bash
kubectl apply -f ./ingress_rules.yaml
```

You just limited the number of concurrent connections from a given client to 10 and it's allowed number of requests per second to 1. You see the power here don't you?

That one wraps it up for today. Hope you enjoyed the ride!

Please download all code and files [here](https://github.com/cmendible/kubernetes.samples) and be sure to check the [online documentation](https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/nginx-configuration/annotations.md) to learn more about the annotations and available features.