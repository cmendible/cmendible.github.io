---
author: Carlos Mendible
categories:
- azure
- devops
date: "2018-05-05T07:21:39Z"
description: Running Ansible Azure playbook in Azure Cloud Shell
image: /assets/img/posts/ansible.png
published: true
# tags: ansible cloudshell
title: Running Ansible Azure playbook in Azure Cloud Shell
---

## **NOTE: The issue described in this post was fixed!!! (ansible 2.5.2 and Azure CLI 2.0.34)**

Last week I tried to run this simple Ansible playbook in Azure Cloud Shell:

``` yaml
# resource_group.yml
# Create test resource group in west europe
- name: Create a Resource Group
  hosts: localhost
  connection: local
  gather_facts: no
  tasks:
    - name: Create Resource Group
      azure_rm_resourcegroup:
        location: westeurope
        name: test
        state: present
```

The first attempt to run the playbook with:

``` shell
ansible-playbook resource_group.yml -vvv
```

failed with the following message: **No module named 'packaging'**

So I ran:

``` shell
pip install --user packaging
```

But this time when I tried to run the playbook I got a message telling me to upgrade the Azure module for Ansible. Again the first attempt didn't go well so I **forced** the upgrade:

``` shell
pip install ansible[azure] --upgrade --force --user
```

Problem solved!!! Runing the following command did create the resource group:

``` shell
ansible-playbook resource_group.yml -vvv
```

Hope it helps!
