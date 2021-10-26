---
author: Carlos Mendible
categories:
- azure
- devops
date: "2018-08-10T00:18:30Z"
description: Installing Azure CLI and Ansible on Ubuntu
images: ["/assets/img/posts/ansible.png"]
published: true
tags: ["ansible", "cli"]
title: Installing Azure CLI and Ansible on Ubuntu
---

I've been using [Ansible](https://www.ansible.com/) and the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/?view=azure-cli-latest) every single day for the last 3 months. Non stop work editing playbooks and scripts with [Visual Studio Code](https://code.visualstudio.com/) and running them on [Ubuntu (WSL)](https://docs.microsoft.com/en-us/windows/wsl/install-win10) on my Windows 10 machine.

Turns out that because [Ansible](https://www.ansible.com/) uses python version **2.7.12** and the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/?view=azure-cli-latest) uses python **3.6.5** you can make a mess if you get "creative" trying to install the tools instead of using the recommended commands:

## 1. Azure CLI
---

To install the Azure CLI just run the following commands:

``` bash
AZ_REPO=$(lsb_release -cs)
echo "deb [arch=amd64] https://packages.microsoft.com/repos/azure-cli/ $AZ_REPO main" | sudo tee /etc/apt/sources.list.d/azure-cli.list

curl -L https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -

sudo apt-get install -y apt-transport-https
sudo apt-get update && sudo apt-get install -y azure-cli
```

If you want to read the official Microsoft documentation please find it [here](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-apt?view=azure-cli-latest)

## 2. Ansible
---

To install Ansible run the following commands:

``` bash
sudo apt-get update && sudo apt-get install -y libssl-dev libffi-dev python-dev python-pip
sudo pip install ansible[azure]
```

Please find the official Microsoft documentation [here](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/ansible-install-configure#ubuntu1604-lts)

## 3. Verify the installation

To confirm that both tools are installed just run:

``` bash
az --version
ansible --verison
```

Take a minute and check the python versions and you'll see the difference!

## 4. Install using a gist
---

I've published a [gist](https://gist.github.com/cmendible/696bc74bfd123924bd255834aeedb340#file-azure_cli_ansible_install-sh) so you can install both Ansible and Azure CLI in one run:

``` bash
sudo curl -s https://gist.githubusercontent.com/cmendible/696bc74bfd123924bd255834aeedb340/raw/a3fd95f7cf6556292a04ee289ae05769f83caa40/azure_cli_ansible_install.sh | bash
```

Hope it helps!!!