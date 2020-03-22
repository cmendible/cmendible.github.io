---
author: Carlos Mendible
categories:
- azure
- dotnetcore
date: "2016-09-26T06:59:08Z"
description: 'Step by step: ASP.NET Core on Docker'
image: /wp-content/uploads/2016/09/dotnetdocker.jpg
tags: ["aspNetCore", "Docker"]
title: 'Step by step: ASP.NET Core on Docker'
url: /2016/09/26/step-by-step-asp-net-core-on-docker/
---
This week I have to give an introductory talk on DevOps and <a href="https://www.docker.com/" target="_blank">Docker</a> and therefore I decided to prepare a simple **Step by step: ASP.NET Core on <a href="https://www.docker.com/" target="_blank">Docker</a>** sample.

Assuming you have <a href="https://www.docker.com/" target="_blank">Docker</a> installed and running, follow these 4 simple steps:

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

## 2. Create a docker image
---
With the dockerfile in place run the following command
    
``` powershell
   sudo docker buildÂ -t hello_world .
```
Now you have an image named <em>hello_world</em> with all the dependencies and code needed to run the sample.

## 3. Test the Docker image
---
To test the Docker image run the following command
    
``` powershell
   docker run -it -p 5000:5000 hello_world
```
    
If everything runs as expected you can head to your browser and navigate to http://localhost:5000 and confirm that the application is running!
         
## 4. Run the Docker image as a daemon process
---
Now that you know that everything is working as expected use the following command to run the Docker image as a daemon process 
          
``` powershell
   docker run -t -d -p 5000:5000 hello_world
```
          
Feel free to logout or close the connection with your Docker box and the application will keep running.

You can get a copy of the docker file here: <a href="https://github.com/cmendible/dotnetcore.samples/tree/master/docker.helloworld">https://github.com/cmendible/dotnetcore.samples/tree/master/docker.helloworld</a>
        
Hope it helps!