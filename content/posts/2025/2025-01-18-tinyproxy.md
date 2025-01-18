---
author: Carlos Mendible
categories:
- azure
date: "2025-01-12"
description: 'tinyproxy: a lightweight HTTP/HTTPS proxy server'
images: ["/assets/img/posts/kubernetes.png"]
draft: false
tags: ["tinyproxy"]
title: 'tinyproxy: a lightweight HTTP/HTTPS proxy server'
---

Tinyproxy is a lightweight HTTP/HTTPS proxy server designed to be fast and small. It is useful for scenarios where you need to set up a proxy server quickly and easily.

Recently I used it to check what happens when a set of Azure domains are blocked (i.e. management.azure.com) and it worked like a charm. 

In this post, we will go through the steps required to install and configure tinyproxy on Ubuntu.

## Install tinyproxy

To install tinyproxy on Ubuntu, run the following commands:

``` bash
sudo apt-get update
sudo apt-get install tinyproxy
```

## Configure tinyproxy

To configure tinyproxy, create a file named `tinyproxy.conf`. You can use the following configuration as a starting point:

``` bash
Port 8080
Listen 127.0.0.1
Timeout 600
Allow 127.0.0.1
Filter "./tinyproxy.filter"
```

## Configure the filter

To configure the filter, create a file named `tinyproxy.filter`. You can use the following configuration as a starting point:

``` bash
# filter the following domains
tiktok.com
```

## Start tinyproxy

To start tinyproxy, run the following command:

``` bash
sudo tinyproxy -d -c tinyproxy.conf
```

## Verify tinyproxy

To verify that tinyproxy is running, run the following curl command:

``` bash
https_proxy=127.0.0.1:8080 http_proxy=127.0.0.1:8080 curl -i https://www.bing.com
```

and to check that it's filtering the domains, run the following curl command:

``` bash
https_proxy=127.0.0.1:8080 http_proxy=127.0.0.1:8080 curl -i https://www.tiktok.com
```

wich should result in a `HTTP/1.1 403 Filtered` response.  

Hope it helps!

**References:**

* [tinyproxy](https://tinyproxy.github.io/)