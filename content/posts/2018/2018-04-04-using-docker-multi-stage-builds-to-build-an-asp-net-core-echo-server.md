---
author: Carlos Mendible
categories:
- dotnet
crosspost_to_medium: true
date: "2018-04-04T15:00:00Z"
description: Using Docker Multi Stage Builds to build an ASP.NET Core Echo Server
images: ["/assets/img/posts/docker.png"]
published: true
tags: ["aspNetCore", "Docker", "Dockerfile"]
title: Using Docker Multi Stage Builds to build an ASP.NET Core Echo Server
url: /2018/04/04/using-docker-multi-stage-builds-to-build-an-asp-net-core-echo-server/
---

Today I'll show you how to create a simple Echo Server with ASP.NET Core and then a Docker Image using [multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/):

## 1. Create the Application
---

Open a PowerShell promt and run:

``` powershell
mkdir echoserver
cd echoserver
dotnet new console
dotnet add package Microsoft.AspNetCore -v 2.0.2
```

## 2. Replace the contents of Program.cs
---

Replace the contents of the **Program.cs** file with the following code:

``` csharp
namespace EchoServer
{
    using Microsoft.AspNetCore.Hosting;
    using Microsoft.AspNetCore.Builder;
    using Microsoft.AspNetCore;

    class Program
    {
        static int Main(string[] args)
        {
            WebHost.CreateDefaultBuilder()
                .UseKestrel()
                .Configure(app =>
                    {
                        app.Run(httpContext =>
                        {
                            var request = httpContext.Request;
                            var response = httpContext.Response;

                            // Echo the Headers
                            foreach (var header in request.Headers)
                            {
                                response.Headers.Add(header);
                            }

                            // Echo the body
                            return request.Body.CopyToAsync(response.Body);
                        });
                    })
                .Build()
                .Run();

            return 0;
        }
    }
}
```

## 3. Build and Test
---

Run the folowing commands to build the application:

``` powershell
dotnet build
dotnet run
```

The Echo Server will be running on: [http://localhost:5000](http://localhost:5000) and to test it just send any request and the server should return it back to you.

Let's try sending a GET request with a custom header:

``` bash
curl -H "Custom-Header: echo" http://localhost:5000/ -i
```

the response should read:

``` bash
HTTP/1.1 200 OK
Date: Wed, 04 Apr 2018 10:10:55 GMT
Server: Kestrel
Content-Length: 0
Accept: */*
Host: localhost:5000
User-Agent: curl/7.35.0
Custom-Header: echo
```

## 4. Create a Dockerfile
---

In order to run the Echo Server in a container create a Dockerfile with the following contents:

``` docker
# This stage builds the application
FROM microsoft/dotnet:2.0.6-sdk-2.1.101-jessie AS builder
COPY . src
WORKDIR src
RUN dotnet restore
RUN dotnet publish -c release

# This stages uses the output from the build
FROM microsoft/dotnet:2.0.6-runtime-jessie
COPY --from=builder src/bin/release/netcoreapp2.0/publish app
WORKDIR app
ENV ASPNETCORE_URLS http://*:80
EXPOSE 80
ENTRYPOINT ["dotnet", "echoserver.dll"]
```

Note: The previous Dockerfile defines a [multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/):

* The first stage copies the source code to the image and builds the application.
* The second stage uses the output of the first stage (builder) to create the final and optimized image (i.e. No need to remove the sdk or the source code).
* Docker 17.05 or higher is needed in order to build and image based on a multi-stage Dockerfile.

## 5. Build and test the image
---

Run the folowing commands to build and run the docker image:

``` powershell
docker build -t echoserver .
docker run -p 5000:80 echoserver
```

To test the image just repeat the tests from step 3.

Hope it helps!

Feel free to get the code [here](https://github.com/cmendible/dotnetcore.samples/tree/main/echoserver).