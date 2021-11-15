---
author: Carlos Mendible
categories:
- azure
- kubernetes
date: "2021-11-09T10:00:00Z"
description: 'AKS: Open Service Mesh Traffic Access Control'
images: ["/assets/img/posts/osm.png"]
draft: false
tags: ["aks", "osm"]
title: 'AKS: Open Service Mesh Traffic Access Control'
---

In my previous post [AKS: Open Service Mesh & mTLS]({{< ref "/posts/2021/2021-11-08-aks-open-service-mesh-mtls.md" >}} ), I described how to deploy an AKS cluster with Open Service Mesh enabled, and how:

* Easy is to onboard applications onto the mesh by enabling automatic sidecar injection of Envoy proxy.
* OSM enables secure service to service communication.

This time I'll show you that [Open Service Mesh (OSM)](https://openservicemesh.io/) also provides a nice feature for controlling traffic between microservices: [Traffic Access Control](https://github.com/servicemeshinterface/smi-spec/blob/main/apis/traffic-access/v1alpha3/traffic-access.md) based on the [SMI specifications](https://github.com/servicemeshinterface/smi-spec).

To control traffic between microservices using OSM you have to be aware of the following custom resource definitions:

* **HTTPRouteGroup**: used to describe HTTP/1 and HTTP/2 traffic, enumerating the routes that can be served by an application.
* **TCPRoute**: used to describe L4 TCP traffic for a list of ports.
* **UDPRoute**: used to describe L4 UDP traffic for a list of ports.
* **TrafficTarget**: associates a set of traffic definitions (rules) with a service identity which is allocated to a group of pods. Access is controlled via referenced TrafficSpecs (HTTPRouteGroup, TCPRoute, UDPRoute) and by a list of source service identities.

> Access is controlled based on service identity, at present the method of assigning service identity is using Kubernetes service accounts, provision for other identity mechanisms will be handled by the spec at a later date.

> A valid TrafficTarget must specify a destination, at least one rule, and at least one source.

Now let's see how this works:

## Deploy an AKS cluster with OSM enabled 

Follow the steps in my previous post [AKS: Open Service Mesh & mTLS]({{< ref "/posts/2021/2021-11-08-aks-open-service-mesh-mtls.md" >}} ) to deploy an AKS cluster with Open Service Mesh enabled.

## Disable Permissive Traffic Policy Mode

By default the OSM service mesh is configured to allow all traffic between services, so let's disable the Permissive Traffic Policy Mode by running:

```	shell
kubectl patch meshconfig osm-mesh-config -n kube-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}' --type=merge
```

Double check that the Permissive Traffic Policy Mode is disabled:

```	shell
kubectl get meshconfig osm-mesh-config -n kube-system -o yaml | grep -i enablePermissiveTrafficPolicyMode
```

## Deploy the sample microservices

### Create nginx.yaml with the following contents:

``` yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: null
  name: nginx
  namespace: default

---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    run: nginx
  annotations:
    openservicemesh.io/sidecar-injection: enabled
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
  serviceAccountName: nginx

---
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  name: nginx
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx
```

**Note that:**

* The `openservicemesh.io/sidecar-injection: enabled` annotation is required to enable sidecar injection.
* The service account `nginx` is used to run the nginx container and will be key when defining the traffic rules.

### Create busybox.yaml with the following contents:

```	yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: null
  name: busybox
  namespace: default

---
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  labels:
    run: busybox
  annotations:
    openservicemesh.io/sidecar-injection: enabled
spec:
  containers:
  - image: busybox
    name: busybox
    ports:
    - containerPort: 80
    command:
    - sh
    - -c
    - sleep 3600
  serviceAccountName: busybox
```

**Note that:**

* The `openservicemesh.io/sidecar-injection: enabled` annotation is required to enable sidecar injection.
* The service account `busybox` is used to run the busybox container and will be key when defining the traffic rules.

### Deploy the microservices:

```	shell
kubectl apply -f nginx.yaml
kubectl apply -f busybox.yaml
```

### Test Connectivity

Since we disabled the Permissive Traffic Policy Mode, traffic is not allowed between the microservices. Test the connectivity by running the following:

```	shell
kubectl exec -it busybox -c busybox -- sh
```

once inside the container, run:

```	shell
wget -O- http://nginx
```

The result should be similar to the following:

```	shell
Connecting to nginx (10.0.149.72:80)
wget: error getting response: Resource temporarily unavailable
```

## Add Traffic Access Control to allow traffic between microservices

Let's fix the issue by adding Traffic Access Control to the mesh and allow communication between the microservices:

### Create nginx_traffic_target.yaml with the following contents:

```	yaml
apiVersion: specs.smi-spec.io/v1alpha4
kind: HTTPRouteGroup
metadata:
  name: all-routes
  namespace: default
spec:
  matches:
  - name: everything
    pathRegex: "/*"
    methods: ["*"]

---
apiVersion: access.smi-spec.io/v1alpha3
kind: TrafficTarget
metadata:
  name: nginx
  namespace: default
spec:
  destination:
    kind: ServiceAccount
    name: nginx
    namespace: default
  rules:
  - kind: HTTPRouteGroup
    name: all-routes
    matches:
    - everything
  sources:
  - kind: ServiceAccount
    name: busybox
    namespace: default
```

**Note that:**

* The `HTTPRouteGroup` `all-routes` allows all traffic and methods, but you could easily make it more restrictive.
* The `TrafficTarget` associates the destination and source with the rules.
* The `TrafficTarget` defines `sources` and `destination` by using service account names: `nginx` and `busybox`.

### Deploy the HTTPRouteGroup and the TrafficTarget

Run:

```	shell
kubectl apply -f nginx_traffic_target.yaml
```

### Recheck Connectivity

Now that we explicitly allowed traffic between the microservices, we can test connectivity by running the following:

```	shell
kubectl exec -it busybox -c busybox -- sh
```

and once inside the container, run the following command to test connectivity:

```	shell
wget -O- http://nginx
```

Hope it helps!!!

Please find the complete sample [here](https://github.com/cmendible/azure.samples/tree/main/aks_osm_smi_traffic_access_control)

References:

* [Open Service Mesh Traffic Access Control](https://github.com/servicemeshinterface/smi-spec/blob/main/apis/traffic-access/v1alpha3/traffic-access.md)
* [Open Service Mesh Traffic Specs](https://github.com/servicemeshinterface/smi-spec/blob/main/apis/traffic-specs/v1alpha4/traffic-specs.md)
* [Open Service Mesh](https://openservicemesh.io/)
* [Open Service Mesh AKS add-on](https://docs.microsoft.com/en-us/azure/aks/open-service-mesh-about)
