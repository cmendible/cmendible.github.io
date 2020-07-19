---
author: Carlos Mendible
categories:
- azure
- dotnet
date: "2016-10-30T15:16:29Z"
description: 'Step by step: Scale ASP.NET Core with Docker Swarm'
images: ["/wp-content/uploads/2016/09/dotnetdocker.jpg"]
tags: ["aspNetCore", "Docker", "Swarm"]
title: 'Step by step: Scale ASP.NET Core with Docker Swarm'
url: /2016/10/30/step-by-step-scale-asp-net-core-with-docker-swarm/
---
A few weeks ago I posted <a href="https://carlos.mendible.com/2016/09/26/step-by-step-asp-net-core-on-docker/" target="_blank">Step by step: ASP.NET Core on Docker</a> were I showed how to build and run a <a href="https://www.docker.com/" target="_blank">Docker</a> image with an ASP.NET Core application.

Today I bring you: **Step by step: Scale ASP.NET Core with Docker Swarm** so you can scale out or in the same [application](https://github.com/cmendible/dotnetcore.samples/tree/main/docker.helloworld).

Assuming you have <a href="https://www.docker.com/" target="_blank">Docker 1.12 or later</a> installed and running, follow this steps:

## 1. Create a dockerfile
---
On your Docker box create a dockerfile with the following contents 
    
``` yaml    
    # We use the microsoft/dotnet image as a starting point.
    FROM microsoft/dotnet

    # Install git
    RUN apt-get install git -y

    # Create a folder to clone our source code 
    RUN mkdir repositories

    # Set our working folder
    WORKDIR repositories

    # Clone the source code
    RUN git clone https://github.com/cmendible/aspnet-core-helloworld.git

    # Set our working folder
    WORKDIR aspnet-core-helloworld/src/dotnetstarter

    # Expose port 5000 for the application.    
    EXPOSE 5000

    # Restore nuget packages
    RUN dotnet restore

    # Start the application using dotnet!!!
    ENTRYPOINT dotnet run
```
## 2. Create a Docker image
---
With the dockerfile in place run the following command 
    
``` powershell
sudo docker build -t hello_world .
```

Now you have an image named <em>hello_world</em> with all the dependencies and code needed to run the sample.  
      
## 3. Initialize a Swarm
---
Initialize <a href="https://docs.docker.com/swarm/" target="_blank">Docker Swarm</a>.
          
``` powershell   
docker swarm init
```   
      
## 4. Create a Docker Service
---
Now that you have setup everything use the following command to create a service named <em>hello_service</em> based on the <em>hello_world</em> image and start it 
          
``` powershell   
docker service create --name hello_service --publish 5000:5000 hello_world
```

Wait a few seconds and navigate to <a href="http://localhost:5000" target="_blank">http://localhost:5000</a> and you should be able to reach the web application.
      
If you want to learn more about Docker services, start here: <a href="https://docs.docker.com/engine/reference/commandline/service_create/" target="_blank">Service Create/</a>
      
## 5. Scale up your application
---
Let's scale your service up to 3 instances with the following command 
          
``` powershell   
docker service scale hello_service=3
```

If you want to see how many replicas your service is running issue the following command:
         
``` powershell   
docker service ls
```

Note that it takes some seconds before all new replicas start.
            
## 6. Scale down your application
---            
Let's scale your service down to 1 instance with the following command 
                
``` powershell   
docker service scale hello_service=1
```
            
## 7. Optional: Remove the service
If you are don't want the service anymore, remove it from the Swarm with the following command 
                
``` powershell   
docker service rm hello_service
```

You can get a copy of the docker file here: <a href="https://github.com/cmendible/dotnetcore.samples/tree/main/docker.helloworld">https://github.com/cmendible/dotnetcore.samples/tree/main/docker.helloworld</a>
        
Hope it helps!    