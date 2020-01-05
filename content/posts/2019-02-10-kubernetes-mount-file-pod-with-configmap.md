---
author: Carlos Mendible
categories:
- kubernetes
- azure
crosspost_to_medium: true
date: "2019-02-10T15:12:00Z"
description: 'Kubernetes: Mount a file in your Pod using a ConfigMap'
image: /assets/img/posts/kubernetes.png
published: true
tags: ["configmap", "volume", "pod"]
title: 'Kubernetes: Mount a file in your Pod using a ConfigMap'
---

Lately I've been learning Go and this week I started a side project named [kube-sherlock](https://github.com/cmendible/kube-sherlock). The purpose of this small program is to list any pod that does not have the **labels** that your organization requires.

For [kube-sherlock](https://github.com/cmendible/kube-sherlock) I created a [dockerfile](https://github.com/cmendible/kube-sherlock/blob/master/dockerfile) were both the program (kube-sherlock) and the default configuration (config.yaml) are placed in the **app** folder:

``` docker
FROM golang:1.11.5 AS build
WORKDIR /src
ADD go.mod go.sum ./
RUN go get -v
ADD kube-sherlock.go config.yaml ./
RUN CGO_ENABLED=0 GOOS=linux go build -a -ldflags '-w'

FROM alpine:3.7
COPY --from=build src/config.yaml app/config.yaml
COPY --from=build src/kube-sherlock app/kube-sherlock
WORKDIR /app
CMD ./kube-sherlock

# Metadata
ARG BUILD_DATE
ARG VCS_REF
LABEL org.label-schema.build-date=$BUILD_DATE \
    org.label-schema.name="kube-sherlock" \
    org.label-schema.description="Check if labels are applied to your containers" \
    org.label-schema.url="https://github.com/cmendible/kube-sherlock" \
    org.label-schema.vcs-ref=$VCS_REF \
    org.label-schema.vcs-url="https://github.com/cmendible/kube-sherlock" \
    org.label-schema.schema-version="0.1"
```

So what if you want to replace the default configuration?

You can achieve this with the help of a **ConfigMap**, creating a new **config.yaml** with your custom values:

``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sherlock-config
  namespace: default
data:
  config.yaml: |
    namespaces:
      - default
    labels:
      - "app"
      - "owner"
```

**Note**: I'm using the name of the file as the **key**.

And then create a pod definition, referencing the **ConfigMap**:

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-sherlock
spec:
  serviceAccountName: kube-sherlock
  containers:
    - name: kube-sherlock
      image: cmendibl3/kube-sherlock:0.1
      volumeMounts:
      - name: config-volume
        mountPath: /app/config.yaml
        subPath: config.yaml
  volumes:
    - name: config-volume
      configMap:
        name: sherlock-config
  restartPolicy: Never
```

**Note**: the volume references the **ConfigMap** (sherlock-config), the volume mount specifies the **mountPath** as the file you want to replace (/app/config.yaml) and the **subPath** property is used to reference the file by key (config.yaml)

Hope it helps.