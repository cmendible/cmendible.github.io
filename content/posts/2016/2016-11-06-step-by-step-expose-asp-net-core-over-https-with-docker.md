---
author: Carlos Mendible
categories:
- azure
- dotnet
date: "2016-11-06T08:00:49Z"
description: 'Step by step: Expose ASP.NET Core over HTTPS with Docker'
images: ["/wp-content/uploads/2016/09/dotnetdocker.jpg"]
tags: ["aspNetCore", "Docker", "https", "Kestrel", "openssl"]
title: 'Step by step: Expose ASP.NET Core over HTTPS with Docker'
url: /2016/11/06/step-by-step-expose-asp-net-core-over-https-with-docker/
---
This week I decided to modify the sample of my previous post: <a href="https://carlos.mendible.com/2016/10/30/step-by-step-scale-asp-net-core-with-docker-swarm/" target="_blank">Step by step: Scale ASP.NET Core with Docker Swarm</a> so you can add TLS to your ASP.NET Core applications and Dockerize it.

Let's see how I changed the application in order to make it work:

## Add HTTPS support for Kestrel
---
I added the following line to the dependencies in the <em>project.json</em> file.
    
``` json 
    "Microsoft.AspNetCore.Server.Kestrel.Https": "1.0.1",
```

## Configure Kestrel to use HTTPS
---
In the <em>Main</em> method I configured Kestrel to use HTTPS. Don't worry about the <em>cert.pfx</em> certificate file because it will be created inside the docker container.
    
Note that in line 8 I also configured the application to use port 443.
    
``` csharp 
    public static void Main(string[] args)
    {
        var host = new WebHostBuilder()
            .UseKestrel((o) => 
            {
                o.UseHttps(new X509Certificate2(@"cert.pfx", Configuration["certPassword"]));
            })
            .UseUrls("https://*:443")
            .UseContentRoot(Directory.GetCurrentDirectory())
            .UseStartup<Startup>()
            .Build();

        host.Run();
    }
```

Now It's time to show you how to Dockerize the application:

## Create a dockerfile
---
  

``` powershell 
# We use the microsoft/dotnet image as a starting point.
FROM microsoft/dotnet 

# Install git
RUN apt-get install git -y

# Clone the source code
RUN git clone -b ssl https://github.com/cmendible/aspnet-core-helloworld.git

# Set our working folder
WORKDIR aspnet-core-helloworld/src/dotnetstarter

# Restore nuget packages
RUN dotnet restore

# Build the application using dotnet!!!
RUN dotnet build

# Set password for the certificate as 1234
# I'm using Environment Variable here to simplify the .NET Core sample.
ENV certPassword 1234

# Use opnssl to generate a self signed certificate cert.pfx with password $env:certPassword
RUN openssl genrsa -des3 -passout pass:${certPassword} -out server.key 2048
RUN openssl rsa -passin pass:${certPassword} -in server.key -out server.key
RUN openssl req -sha256 -new -key server.key -out server.csr -subj '/CN=localhost'
RUN openssl x509 -req -sha256 -days 365 -in server.csr -signkey server.key -out server.crt
RUN openssl pkcs12 -export -out cert.pfx -inkey server.key -in server.crt -certfile server.crt -passout pass:${certPassword}

# Expose port 443 for the application.
EXPOSE 443

# Start the application using dotnet!!!
ENTRYPOINT dotnet run
```

## Create a Docker image
---
With the dockerfile in place run the following command 
    
``` powershell
sudo docker build -t httpssample .
```

Now you have an image named <em>httpssample</em> with all the dependencies and code needed to run the application.
            
## Test the Docker image
To test the Docker image run the following command 
          
``` powershell
sudo docker run -it -p 443:443 httpssample 
```

Browse to <a href="https://localhost" target="_blank">https://localhost</a> Your browser will warn about the certificate because it's self signed.
            
## Run the Docker image as a daemon process

Now that you know that everything is working as expected use the following command to run the Docker image as a daemon process 
                
``` powershell
docker run -t -d -p 443:443 httpssample 
```
            
You can get a copy of the docker file here: <a href="https://github.com/cmendible/dotnetcore.samples/tree/main/docker.helloworld.https">https://github.com/cmendible/dotnetcore.samples/tree/main/docker.helloworld.https</a>
     
        
Hope it helps!