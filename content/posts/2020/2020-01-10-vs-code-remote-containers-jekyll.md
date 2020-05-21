---
author: Carlos Mendible
categories:
- devops
crosspost_to_medium: true
date: "2020-01-10T00:00:00Z"
description: 'Visual Studio Code Remote Containers: Jekyll'
images: ["/assets/img/posts/code.png"]
published: true
tags: ["jekyll", "Docker"]
title: 'Visual Studio Code Remote Containers: Jekyll'
url: /2020/01/10/vs-code-remote-containers-jekyll/
---

For the last 3 years this blog was written using [Jekyll](https://jekyllrb.com/docs/installation/) which has a series of requirements such as Ruby that I don't want to keep installing or maintaining on my PC. So I created this Developer Container for those who want to use Jekyll from an isolated container.

Let's check the container definition:

## 1. devcontainer.json file
---

This file configures the remote container with the specified extensions

``` json
{
	"name": "Jekyll",
	"dockerFile": "Dockerfile",

	// Use 'settings' to set *default* container specific settings.json values on container create. 
	// You can edit these settings after create using File > Preferences > Settings > Remote.
	"settings": { 
		"terminal.integrated.shell.linux": "/bin/bash"
	},

	// Use 'appPort' to create a container with published ports. If the port isn't working, be sure
	// your server accepts connections from all interfaces (0.0.0.0 or '*'), not just localhost.
	// "appPort": ["3000:3000"],

	// Uncomment the next line to run commands after the container is created.
	// "postCreateCommand": "cd ${input:projectName} && bundle install",

	// Uncomment the next line to use a non-root user. On Linux, this will prevent
	// new files getting created as root, but you may need to update the USER_UID
	// and USER_GID in .devcontainer/Dockerfile to match your user if not 1000.
	// "runArgs": [ "-u", "vscode" ],

	// Add the IDs of extensions you want installed when the container is created in the array below.
	"extensions": []
}
```

## 2. Dockerfile
---

This is the Dockerfile with all the tooling for the Jekyll environment.

``` Dockerfile
FROM debian:latest

# Avoid warnings by switching to noninteractive
ENV DEBIAN_FRONTEND=noninteractive

# This Dockerfile adds a non-root 'vscode' user with sudo access. However, for Linux,
# this user's GID/UID must match your local user UID/GID to avoid permission issues
# with bind mounts. Update USER_UID / USER_GID if yours is not 1000. See
# https://aka.ms/vscode-remote/containers/non-root-user for details.
ARG USERNAME=vscode
ARG USER_UID=1000
ARG USER_GID=$USER_UID

# Configure apt and install packages
RUN apt-get update \
    && apt-get -y install --no-install-recommends apt-utils dialog 2>&1 \
    #
    # Install vim, git, process tools, lsb-release
    && apt-get install -y \
        git \
    #
    # Install ruby
    && apt-get install -y \
        make \
		build-essential \
		ruby \
        ruby-dev \
    #
    # Install jekyll
    && gem install \
        bundler \
        jekyll \
    #
    # Create a non-root user to use if preferred - see https://aka.ms/vscode-remote/containers/non-root-user.
    && groupadd --gid $USER_GID $USERNAME \
    && useradd -s /bin/bash --uid $USER_UID --gid $USER_GID -m $USERNAME \
    # [Optional] Add sudo support for the non-root user
    && apt-get install -y sudo \
    && echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME\
    && chmod 0440 /etc/sudoers.d/$USERNAME \
    #
    # Clean up
    && apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*

# Switch back to dialog for any ad-hoc use of apt-get
ENV DEBIAN_FRONTEND=dialog

```

To learn more about this Remote Container please check [here](https://github.com/microsoft/vscode-dev-containers/tree/master/containers/azure-blockchainhttps://github.com/microsoft/vscode-dev-containers/tree/master/containers/jekyll).

Hope it helps!
