---
author: Carlos Mendible
categories:
- dotnet
crosspost_to_medium: true
date: "2017-07-26T09:55:19Z"
description: Run ASP.NET Core on OpenShift
images: ["/wp-content/uploads/2017/07/openshift"]
tags: ["aspNetCore", "Docker", "Dockerfile", "OpenShift"]
title: Run ASP.NET Core on OpenShift
---
Today I'll show you how to **Run ASP.NET Core on OpenShift**.

First be aware of the following prerequisites:

  * You'll need a working Docker installation. If you are using Windows 10 you can get Docker for Windows <a href="https://docs.docker.com/docker-for-windows/install/#download-docker-for-windows" target="_blank">here</a>.
  * Be sure to **Disable TLS for 127.0.0.1:2375** for your Docker installation
  * Be sure to set **172.30.0.0/16 as an Insecure Registry** in your Docker installation
  * Be sure to set **172.30.1.1:5000 as a Registry Mirror** in your Docker installation
  * You'll need download the <a href="https://www.openshift.org/download.html" target="_blank">oc Client Tools</a>

**Note**: If you want to learn about OpenShift and what can you do with it, I recommend the free book: <a href="https://www.openshift.com/promotions/for-developers.html" target="_blank">Openshift for Developers</a>.

Now let's start:

## 1. Create a folder for your new project
---
Open a command promt an run:
    
``` powershell
mkdir aspnet.on.openshift
```

## 2. Create the project
---

``` powershell
cd aspnet.on.openshift
dotnet new web
```
No you have a working Hello World application.

## 3. Publish your application
---
Restore the nuget packages and publish your application with the following commands:
          
``` powershell
dotnet restore
dotnet publish -c release
```

## 4. Create a Dockerfile
---
Create a Dockerfile (be aware of the capital D) with the following contents:

``` yml
FROM microsoft/dotnet 

# Set ASPNETCORE_URLS
ENV ASPNETCORE_URLS=https://*:8080

# Switch to root for changing dir ownership/permissions
USER 0

# Copy the binaries
COPY /bin/release/netcoreapp1.1/publish app

# Change to app directory
WORKDIR app

# In order to drop the root user, we have to make some directories world
# writable as OpenShift default security model is to run the container under
# random UID.
RUN chown -R 1001:0 /app && chmod -R og+rwx /app

# Expose port 8080 for the application.
EXPOSE 8080

# Run container by default as user with id 1001 (default)
USER 1001

# Start the application using dotnet!!!
ENTRYPOINT dotnet aspnet.on.openshift.dll
```

## 5. Create you OpenShift Cluster
---
Run the following command:

``` powershell
oc cluster up --host-data-dir=/mydata --version=v1.5.1
```

You should have a working cluster. To get the url of your cluster run:
          
``` powershell
oc status 
```
          
Browse to your cluster (i.e <a href="https://10.0.75.2:8443" target="_blank">https://10.0.75.2:8443</a>) and login with username: **developer** and password **developer**

## 6. Create an app in OpenShift
---
Run the following command: 
          
``` powershell
oc new-app . --name=aspnetoc
```
 
## 7. Build your OpenShift image
---
Run the following command 
          
``` powershell
oc start-build aspnetoc --from-dir=.
```

## 8. Create a route so you can access the application
--- 
Run the following commands 
          
``` powershell
oc create route edge --service=aspnetoc
oc get route aspnetoc
```
          
Copy the host/port output of the previous command (i.e. aspnetoc-myproject.10.0.75.2.nip.io)

## 9. Check the status and browse to you application
---
Check the status with the following command 
          
``` powershell
oc status
```
          
When your pod has been deployed browse to your application using the host you copied in step 8 (i.e. <a href="https://aspnetoc-myproject.10.0.75.2.nip.io" target="_blank">https://aspnetoc-myproject.10.0.75.2.nip.io</a>).

Enjoy! you just deployed your ASP.NET Core application to OpenShift.

You can get the code <a href="https://github.com/cmendible/dotnetcore.samples/tree/main/aspnet.on.openshift" target="_blank">here</a>.

Hope it helps!