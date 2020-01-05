---
author: Carlos Mendible
categories:
- dotnetcore
crosspost_to_medium: true
date: "2018-09-25T20:00:00Z"
description: Revisiting Docker Multi Stage Builds to build an ASP.NET Core Echo Server
image: /assets/img/posts/docker.png
published: true
tags: ["aspNetCore", "Docker", "Dockerfile"]
title: Revisiting Docker Multi Stage Builds to build an ASP.NET Core Echo Server
---

On April I wrote a post about [Using Docker Multi Stage Builds to build an ASP.NET Core Echo Server]({{ site.baseurl }}{% post_url 2018-04-04-using-docker-multi-stage-builds-to-build-an-asp-net-core-echo-server %}) and these days while preparing a talk, on CI/CD and kubernetes, I started to play with the [simple sample](https://github.com/cmendible/dotnetcore.samples/tree/master/echoserver) I wrote back then.

Soon enough I noticed that with each ```docker build``` command I issued the dependencies for the echoserver were always restored, even if no changes were made to the project file (csproj).

Let see what was going on:

This was the original dockerfile:

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

As you can see the culprit was between the lines 3 and 5. I was restoring the dependencies after copying all the code! And I fixed it with the following new dockerfile:

``` docker
FROM microsoft/dotnet:2.0.6-sdk-2.1.101-jessie AS builder
COPY ./*.csproj ./src/
WORKDIR /src
RUN dotnet restore
COPY . /src
RUN dotnet publish -c release

FROM microsoft/dotnet:2.0.6-runtime-jessie
COPY --from=builder src/bin/release/netcoreapp2.0/publish app
WORKDIR app
ENV ASPNETCORE_URLS http://*:80
EXPOSE 80
ENTRYPOINT ["dotnet", "echoserver.dll"]
```

Now the dependencies are restored after copying just the project files (*.csproj) then we copy the rest of the files and build as usual, and yes! with this dockerfile in place the image build times are faster if no dependencies are changed!

Hope it helps!

Feel free to get the code [here](https://github.com/cmendible/dotnetcore.samples/tree/master/echoserver.improved).