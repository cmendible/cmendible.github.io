---
author: Carlos Mendible
categories:
- azure
- devops
crosspost_to_medium: true
date: "2019-07-06T00:00:00Z"
description: 'Visual Studio Code Remote Containers: Azure Blockchain'
images: ["/assets/img/posts/code.png"]
published: true
tags: ["blockchain", "truffle", "ganache", "nodejs", "ethereum", "Docker"]
title: 'Visual Studio Code Remote Containers: Azure Blockchain'
url: /2019/07/06/vs-code-remote-containers-azure-blockchain/
---

After collaborating with the [Azure Ansible](https://github.com/microsoft/vscode-dev-containers/tree/master/containers/azure-ansible) container I decided to also develop a Developer Container for those who want or need to use the [Azure Blockchain Development Kit for Ethereum](https://marketplace.visualstudio.com/items?itemName=AzBlockchain.azure-blockchain) to create smart contracts, taking away the burden of installing Python, Truffle, Ganache and NodeJS on your machine.

Once again I collaborated with [Chuck Lantz](http://chuxel.github.io/) and the container definition resulted in the following two files:

## 1. devcontainer.json file
---

This file configures the remote container with the specified extensions

``` json
{
	"name": "Azure Blockchain",
	"dockerFile": "Dockerfile",
	// Uncomment the next line if you will use a ptrace-based debugger like C++, Go, and Rust.
	// "runArgs": [ "--cap-add=SYS_PTRACE", "--security-opt", "seccomp=unconfined" ],
	// Uncomment the next line if you want to publish any ports.
	// "appPort": [],
	// Uncomment the next line if you want to add in default container specific settings.json values
	// "settings":  { "workbench.colorTheme": "Quiet Light" },
	// Uncomment the next line to run commands after the container is created.
	// "postCreateCommand": "az --version",
	"extensions": [
		"ms-vscode.azurecli",
		"azblockchain.azure-blockchain"
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

FROM python:2.7

# Avoid warnings by switching to noninteractive
ENV DEBIAN_FRONTEND=noninteractive

# Configure apt and install packages
RUN apt-get update \
    && apt-get -y install --no-install-recommends apt-utils 2>&1 \
    #
    # Verify git, process tools installed
    && apt-get -y install git procps \
    #
    # Install nodejs
    && curl -sL https://deb.nodesource.com/setup_10.x | bash - \
    && apt-get install -y nodejs \
    #
    # Install Truffle Suite
    && npm i --unsafe-perm -g truffle \
    #
    # Install Ganache CLI
    && npm install -g ganache-cli \
    # 
    # Install the Azure CLI
    && apt-get install -y apt-transport-https curl gnupg2 lsb-release \
    && echo "deb [arch=amd64] https://packages.microsoft.com/repos/azure-cli/ $(lsb_release -cs) main" > /etc/apt/sources.list.d/azure-cli.list \
    && curl -sL https://packages.microsoft.com/keys/microsoft.asc | apt-key add - 2>/dev/null \
    && apt-get update \
    && apt-get install -y azure-cli \
    #
    # Clean up
    && apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*

# Switch back to dialog for any ad-hoc use of apt-get
ENV DEBIAN_FRONTEND=dialog

```

To learn more about this Remote Container please check [here](https://github.com/microsoft/vscode-dev-containers/tree/master/containers/azure-blockchain).

If you want to use Developer Conatianers with WSL 2 start [here](https://docs.microsoft.com/en-us/windows/wsl/tutorials/wsl-containers?WT.mc_id=DOP-MVP-5002618).

Hope it helps!
