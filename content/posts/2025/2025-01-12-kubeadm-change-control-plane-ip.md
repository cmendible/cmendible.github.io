---
author: Carlos Mendible
categories:
- kubernetes
date: "2025-01-12"
description: 'kubeadm: change the control plane IP'
images: ["/assets/img/posts/aks.png"]
draft: false
tags: ["kubeadm", "kubernetes"]
title: 'kubeadm: change the control plane IP'
---

In this post, we will go through the steps required to change the control plane IP of a Kubernetes cluster managed by kubeadm. This can be useful in scenarios where the IP address of the control plane node needs to be updated due to network changes or other reasons.

To change the control plane IP of your Kubernetes cluster, follow these steps:

1. **Stop kubelet service**: Stop the kubelet service to make changes to the control plane.
1. **Remove certificates and configuration files**: Delete all existing certificates and configuration files to reset the control plane.
1. **Update static pod manifests**: Replace the old IP with the new IP in all static pod manifests.
1. **Update kubelet configuration**: Update the IP in the kubelet configuration file.
1. **Stop control plane containers**: Stop all running control plane containers.
1. **Initialize control plane with new IP**: Reinitialize the control plane with the new IP using `kubeadm init`.
1. **Verify new IP in certificate**: Check the new IP in the apiserver certificate to ensure it has been updated.
1. **Wait for apiserver to start**: Wait until the apiserver container is running.
1. **Wait for cluster readiness**: Ensure the cluster is ready by checking the status of the pods.
1. **Restart coredns and network plugin pods**: Restart the coredns and network plugin pods to apply the new configuration.
1. **Update kubectl configuration**: Copy the new kubectl configuration to the user's home directory and set the appropriate permissions.

## Script to change control plane IP

Create a script with the following content to automate the process of changing the control plane IP:

``` bash
# Check if the correct number of arguments are provided
if [ "$#" -ne 2 ]; then
    echo "Usage: $0 <oldIP> <newIP>"
    exit 1
fi

# Assign the script arguments to variables
oldIP=$1
newIP=$2

echo "Stopping kubelet service"
systemctl stop kubelet

echo "Removing al certificates and *.conf files"
rm -rf /etc/kubernetes/pki/*
rm /var/lib/kubelet/pki/kubelet*
rm /etc/kubernetes/*.conf

echo "Changing IP from $oldIP to $newIP for all static pod manifests"
find /etc/kubernetes -type f | xargs grep $oldIP
find /etc/kubernetes -type f | xargs sed -i "s/$oldIP/$newIP/"

echo "Changing IP from $oldIP to $newIP for kubelet configuration"
find /etc/default/kubelet -type f | xargs sed -i "s/$oldIP/$newIP/"

echo "Stoping all controlplane containers"
crictl stop -t 0 $(crictl ps -aq)

echo "Initializing the controlplane with new IP"
# Keep the updated static pod manifests and etcd data
kubeadm init --control-plane-endpoint $newIP \
    --ignore-preflight-errors=FileAvailable--etc-kubernetes-manifests-kube-apiserver.yaml \
    --ignore-preflight-errors=FileAvailable--etc-kubernetes-manifests-kube-controller-manager.yaml \
    --ignore-preflight-errors=FileAvailable--etc-kubernetes-manifests-kube-scheduler.yaml \
    --ignore-preflight-errors=FileAvailable--etc-kubernetes-manifests-etcd.yaml \
    --ignore-preflight-errors=DirAvailable--var-lib-etcd

echo "Showing the IP present in the new apiserver certificate:"
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep -A 1 "Subject Alternative Name"

# Wait until the apiserver container is running
while ! crictl ps | grep -q "apiserver"; do
    echo "Waiting for the apiserver container to start..."
    sleep 5
done

export KUBECONFIG=/etc/kubernetes/admin.conf

# Wait until the kubectl command can successfully get the pods
until kubectl get po -A > /dev/null 2>&1; do
    echo "Waiting for the cluster to be ready..."
    sleep 5
done

echo "Restarting all coredns pods"
kubectl --namespace kube-system delete pods --selector k8s-app=kube-dns --force --grace-period=0

echo "Restarting all calico pods (network plugin)"
kubectl --namespace kube-system delete pods --selector k8s-app=calico-kube-controllers --force --grace-period=0

echo "-------------------------------------------------------------"

echo "The new kubectl configuration is:"
echo "/etc/kubernetes/admin.conf" 
echo "-------------------------------------------------------------"
echo "To use the new kubectl configuration, run the following commands:"
echo "mkdir -p $HOME/.kube"
echo "sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config"
echo "sudo chown $(id -u):$(id -g) $HOME/.kube/config"

echo "-------------------------------------------------------------"
echo "Done"
```

By following the steps outlined above, you can successfully change the control plane IP of your kubeadm Kubernetes cluster. Make sure to verify the new configuration and ensure that all components are working as expected.

> Note: This script is provided as a reference and may need to be modified based on your specific requirements. Make sure you've backed up your cluster and don't forget to reconfigure your cluster nodes.

Hope it helps!

**References:**

* [kubeadm init](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/)
* [How To Setup Kubernetes Cluster Using Kubeadm](https://devopscube.com/setup-kubernetes-cluster-kubeadm/)