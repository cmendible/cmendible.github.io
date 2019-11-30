---
title: "AKS: Read Azure Key Vault secrets using AAD Pod Identity"
published: true
ate: 2019-11-30T07:00:00+00:00
author: Carlos Mendible
layout: post
image: /assets/img/posts/aks.png
description: "AKS: Read Azure Key Vault secrets using AAD Pod Identity"
crosspost_to_medium: true
categories:
  - Azure
  - kubernetes
tags: aks keyvault
---

What if I tell you that it's possible to connect you AKS pods to an **Azure Key Vault** using identities but without having to use credentials in an explicit way?

Well with **AAD Pod Identities** you can enable your Kubernetes applications to access Azure cloud resources securely using **Azure Active Directory** (AAD) including **Azure Key Vault**.

The following gist show a **PowerShell** script that will help you setup everything inside your RBAC enabled AKS cluster. You will also need to have **Azure CLI** installed on your box and an **Azure Key Vault** deployed in the same resource group where your AKS lives.

{% gist 4dd5c48a45480d5a0db0d6ed68e4cd01 %}

The script creates a Manged Identity, assigns some permissions to it and creates a policy inside the Key Vault enabling the Identity to list and get secrets.

Then the **[Managed Identity Controller (MIC) deployment and the Node Managed Identity (NMI) daemon set](https://github.com/Azure/aad-pod-identity#components)** are deployed inside the cluster.

In the last step, two resources are deployed. The first one is an **AzureIdentity** that will be used to identify the Managed Identity inside your cluster and the second one is an **AzureIdentityBinding** that binds the azure Identity with a Selector.

Let's run the powershell command with the following parameters:

* **Resourece Group**: myResourceGroup
* **Managed Identity Name**: myId
* **Identity Selector**: requires-vault
* **AKS Name**: myAKS
* **Key Vault Name**: myKeyVault

``` powershell
.\SetupPodIdentityKeyVaultIntegration.ps1 myResourceGroup myId requires-vault myAKS myKeyVault
```

Once the command is done, any pod marked with the ***aadpodidbinding: requires-vault*** label will get an Identity assigned.

To check that everything is working as expected you can create a **deployment.yaml** with the following contents:

``` yaml
---
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: az-keyvault-reader-test
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: az-keyvault-reader-test
        aadpodidbinding: requires-vault
    spec:
      containers:
        - name: busybox
          image: busybox
          command:
            [
              "sh",
              "-c",
              "wget --tries=70 -qO- http://127.0.0.1:8333/secrets/<SECRET_NAME>/",
            ]
          imagePullPolicy: Always
          resources:
            requests:
              memory: "4Mi"
              cpu: "100m"
            limits:
              memory: "8Mi"
              cpu: "200m"
        - name: az-keyvault-reader-sidecar
          image: cmendibl3/az-keyvault-reader:0.2
          imagePullPolicy: Always
          env:
            - name: AZURE_KEYVAULT_NAME
              value: <AZURE_KEYVAULT_NAME>
          resources:
            requests:
              memory: "8Mi"
              cpu: "100m"
            limits:
              memory: "16Mi"
              cpu: "200m"
      restartPolicy: Always
```

**Note**: Replace the values for <AZURE_KEYVAULT_NAME> with the name of your Key Vault and <SECRET_NAME> with the name of an existing secret stored in your Key Vault:

Now deploy to Kubernetes:

``` powershell
kubectl apply -f ./deployment.yaml
```

and check the logs for the **busybox** pod:

``` powershell
kubectl logs -f $(kubectl get po --selector=app=az-keyvault-reader-test -o jsonpath='{.items[*].metadata.name}') -c busybox -w
```

If everything is OK you should see the value of your secret dumped in the logs (Bad security practice here)!!! And yes you did all this without knowing any credentials!

Please  find the code used to connect to the **Azure KeyVault** here: [az-keyvault-reader.go](https://github.com/cmendible/az-keyvault-reader/blob/master/src/az-keyvault-reader.go) and check [AAD Pod Identity](https://github.com/Azure/aad-pod-identity) for more information on how this "magic" works.

Hope it helps!