---
author: Carlos Mendible
categories:
- azure
- devops
crosspost_to_medium: true
date: "2019-07-05T23:30:00Z"
description: 'Visual Studio Code Remote Containers: Azure Ansible'
images: ["/assets/img/posts/code.png"]
published: true
tags: ["ansible", "Docker"]
title: 'Visual Studio Code Remote Containers: Azure Ansible'
url: /2019/07/05/vs-code-remote-containers-azure-ansible/
---

Last year I was working on a project for deploying Azure services using Ansible, and let me tell you something: Back then a feature like [Visual Studio Remote Containers](https://github.com/microsoft/vscode-dev-containers) would have helped us so much!

Why? Because just installing [Visual Studio Code](https://code.visualstudio.com/), the [Remote Development Extension Pack](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack), and [Docker](https://www.docker.com/products/docker-desktop) you have a killer combo that makes it possible to create a Development environment in a snap and share it with your source code.

When I learned about this feature I tought I should create a Developer Container for those who need to work with Azure and Ansible so I went hands on and after collaborating with [Chuck Lantz](http://chuxel.github.io/) the container definition resulted in the following two files:

## 1. devcontainer.json
---

This file configures the remote container with the specified extensions

``` json
{
	"name": "Azure Ansible",
	"dockerFile": "Dockerfile",
	"runArgs": [
		// Uncomment the next line if you will use a ptrace-based debugger like C++, Go, and Rust.
		// "--cap-add=SYS_PTRACE", "--security-opt", "seccomp=unconfined",
		"-v", "/var/run/docker.sock:/var/run/docker.sock"
	],

	// Uncomment the next line if you want to publish any ports.
	// "appPort": [],

	// Uncomment the next line if you want to add in default container specific settings.json values
	// "settings":  { "workbench.colorTheme": "Quiet Light" },

	// Uncomment the next line to run commands after the container is created.
	// "postCreateCommand": "ansible --version",

	"extensions": [
		"vscoss.vscode-ansible",
		"redhat.vscode-yaml",
		"ms-vscode.azurecli"
	]
}
```

## 2. Dockerfile
---

This is the Dockerfile with all the tooling for the Development environment.

``` Dockerfile
#-------------------------------------------------------------------------------------------------------------
# Copyright (c) Microsoft Corporation. All rights reserved.
# Licensed under the MIT License. See https://go.microsoft.com/fwlink/?linkid=2090316 for license information.
#-------------------------------------------------------------------------------------------------------------

# Pick any base image, but if you select node, skip installing node. 😊
FROM debian:9

# Avoid warnings by switching to noninteractive
ENV DEBIAN_FRONTEND=noninteractive

# Configure apt and install packages
RUN apt-get update \
    && apt-get -y install --no-install-recommends apt-utils 2>&1 \
    #
    # Verify git, required tools installed
    && apt-get install -y \
        git \
        curl \
        procps \
        unzip \
        apt-transport-https \
        ca-certificates \
        gnupg-agent \
        software-properties-common \
        lsb-release 2>&1 \
    #
    # [Optional] Install Node.js for Azure Cloud Shell support 
    # Change the "lts/*" in the two lines below to pick a different version
    && curl -so- https://raw.githubusercontent.com/creationix/nvm/v0.34.0/install.sh | bash 2>&1 \
    && /bin/bash -c "source $HOME/.nvm/nvm.sh \
        && nvm install lts/* \
        && nvm alias default lts/*" 2>&1 \
    #
    # [Optional] For local testing instead of cloud shell
    # Install Docker CE CLI.
    && curl -fsSL https://download.docker.com/linux/$(lsb_release -is | tr '[:upper:]' '[:lower:]')/gpg | apt-key add - 2>/dev/null \
    && add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/$(lsb_release -is | tr '[:upper:]' '[:lower:]') $(lsb_release -cs) stable" \
    && apt-get update \
    && apt-get install -y docker-ce-cli \
    #
    # [Optional] For local testing instead of cloud shell
    # Install the Azure CLI
    && echo "deb [arch=amd64] https://packages.microsoft.com/repos/azure-cli/ $(lsb_release -cs) main" > /etc/apt/sources.list.d/azure-cli.list \
    && curl -sL https://packages.microsoft.com/keys/microsoft.asc | apt-key add - 2>/dev/null \
    && apt-get update \
    && apt-get install -y azure-cli \
    #
    # Install Ansible
    && apt-get install -y libssl-dev libffi-dev python-dev python-pip \
    && pip install ansible[azure] \
    #
    # Clean up
    && apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*

# Switch back to dialog for any ad-hoc use of apt-get
ENV DEBIAN_FRONTEND=dialog
```

To learn more about the Azure Ansible Remote Container please check [here](https://github.com/microsoft/vscode-dev-containers/tree/master/containers/azure-ansible).

Hope it helps!
