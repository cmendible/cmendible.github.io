---
author: Carlos Mendible
categories:
- kubernetes
crosspost_to_medium: true
date: "2021-05-09T17:00:00Z"
description: 'Running k3s inside WSL2 on my ARM64 Surface Pro X'
images: ["/assets/img/posts/kubernetes.png"]
draft: true
tags: ["k3s", "arm64"]
title: 'Running k3s inside WSL2 on my ARM64 Surface Pro X'
---

I'm a proud owner of a Surafe Pro X SQ2 which is an ARM64 device. If you've been reading me, you know I like to tinker with kubernetes and therefore I needed a solution for this device.

I remembered reading about [k3s](https://k3s.io/) a lightweight kubernetes distro built for IoT & Edge computing, and decided to  give it a try.

## Installing k3s in WSL2

### Download the binaries

``` shell
wget https://github.com/k3s-io/k3s/releases/download/v1.19.10%2Bk3s1/k3s-arm64
```

### Rename the file

``` shell
mv k3s-arm64 k3s
```

### Make the file executable

``` shell
chmod +x k3s
```

### Move the file to `/usr/local/bin`

``` shell
sudo mv k3s /usr/local/bin
```

### Copy to k3s config file to `$HOME/.kube`

``` shell
sudo cp /etc/rancher/k3s/k3s.yaml $HOME/.kube
```

### Make current user owner of the k3s config file

``` shell
sudo chown $USER $HOME/.kube/k3s.yaml
```

## Running k3s

### Run the k3s kubernetes server

``` shell
sudo k3s server
```

## Test k3s

From another WSL2 console window: 

### Add the k3s config file to KUBECONFIG environment variable

``` shell
export KUBECONFIG=$HOME/.kube/config:$HOME/.kube/k3s.yaml
```

### Use k3s context

```
kubectl config use-context default
```

### Check all running pods

``` shell
kubectl get po --all-namespaces
```

you should get an output similar to this one:

``` shell
NAMESPACE     NAME                                     READY   STATUS      RESTARTS   AGE
kube-system   helm-install-traefik-mhhfz               0/1     Completed   0          20d
kube-system   metrics-server-7b4f8b595-7cfsb           1/1     Running     1          20d
kube-system   svclb-traefik-pqn56                      2/2     Running     2          20d
kube-system   coredns-66c464876b-pnpgm                 1/1     Running     1          20d
kube-system   local-path-provisioner-7ff9579c6-nhnbj   1/1     Running     6          20d
kube-system   traefik-5dd496474-94lt5                  1/1     Running     1          20d
```

> k3s uses: `local-path-provisioner` and saves volume data in the `/var/lib/rancher/k3s/data` folder

Hope it helps!!!