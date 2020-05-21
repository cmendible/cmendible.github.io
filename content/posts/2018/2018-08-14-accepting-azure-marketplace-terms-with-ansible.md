---
author: Carlos Mendible
categories:
- azure
- devops
crosspost_to_medium: true
date: "2018-08-14T11:27:39Z"
description: Accepting Azure Marketplace Terms with Ansible
images: ["/assets/img/posts/ansible.png"]
published: true
tags: ["ansible"]
title: Accepting Azure Marketplace Terms with Ansible
---

Last May I wrote: [Accepting Azure Marketplace Terms with Azure CLI](https://carlos.mendible.com/2018/05/21/accepting-azure-marketplace-terms-with-azure-cli) and this time we'll accomplish the same task with Ansible.

Turns out that Ansible 2.6 comes with a handy new module: [azure_rm_resource](https://docs.ansible.com/ansible/latest/modules/azure_rm_resource_module.html?highlight=azure_rm_resource) which lets you create, update or delete any Azure resource using Azure REST API. So I decided to take it for a test drive with the "Accepting Terms" sample.

I've hard coded Palo Alto Networks offer for the following playbook but feel free to use another offer:

``` yaml
---
- hosts: localhost
  gather_facts: no
  vars:
    subscriptionId: "[REPLACE WITH YOUR OWN VALUES]"
    az_client_id: "[REPLACE WITH YOUR OWN VALUES]"
    az_tenant: "[REPLACE WITH YOUR OWN VALUES]"
    az_secret: "[REPLACE WITH YOUR OWN VALUES]"
    region: "westeurope"
    publisher: "paloaltonetworks"
    offer: "vmseries1"
    sku: "bundle1"
  tasks:
    - name: Get Agreements for given Image parameters
      azure_rm_resource:
        subscription_id: "{{ subscriptionId }}"
        client_id: "{{ az_client_id }}"
        tenant: "{{ az_tenant }}"
        secret: "{{ az_secret }}"
        url: "https://management.azure.com/subscriptions/{{ subscriptionId }}/providers/Microsoft.MarketplaceOrdering/offerTypes/virtualmachine/publishers/{{ publisher }}/offers/{{ offer }}/plans/{{ sku }}/agreements/current?api-version=2015-06-01"
        method: GET
        api_version: "2015-06-01"
      register: agreement_result

    - name: Register Agreement properties as fact
      set_fact:
        agreement: "{{ agreement_result.response.properties }}"

    - name: Accept Terms for given Image parameters
      azure_rm_resource:
        subscription_id: "{{ subscriptionId }}"
        client_id: "{{ az_client_id }}"
        tenant: "{{ az_tenant }}"
        secret: "{{ az_secret }}"
        url: "https://management.azure.com/subscriptions/{{ subscriptionId }}/providers/Microsoft.MarketplaceOrdering/offerTypes/virtualmachine/publishers/{{ publisher }}/offers/{{ offer }}/plans/{{ sku }}/agreements/current?api-version=2015-06-01"
        method: PUT
        api_version: "2015-06-01"
        body:
          properties:
            publisher: "{{agreement.publisher}}"
            product: "{{agreement.product}}"
            plan: "{{agreement.plan}}"
            licenseTextLink: "{{agreement.licenseTextLink}}"
            privacyPolicyLink: "{{agreement.privacyPolicyLink}}"
            retrieveDatetime: "{{agreement.retrieveDatetime}}"
            signature: "{{agreement.signature}}"
            accepted: "true"
      register: result

    - name: Accept Terms output
      debug:
        msg: "{{result}}"
```

Hope it helps!
