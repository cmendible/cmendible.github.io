---
author: Carlos Mendible
categories:
- kubernetes
- dotnet
crosspost_to_medium: true
date: "2020-03-22T00:00:00Z"
description: 'Reading Kubernetes Secrets with Dapr and .NET Core'
images: ["/assets/img/posts/aks.png"]
published: true
tags: ["dapr"]
title: 'Reading Kubernetes Secrets with Dapr and .NET Core'
---

[Dapr](https://dapr.io) is an event-driven, portable runtime for building microservices on cloud and edge. 

[Dapr](https://dapr.io) supports the fundamental features you'll need such as: service invocation, state management, publish/subscribe messaging and since version [0.5.0](https://github.com/dapr/dapr/releases/tag/v0.5.0) the ability to read from secret stores!

This post will show you to read kubernetes secrets using [Dapr](https://dapr.io) and .NET Core:

## Prerequistes
* [.Net Core SDK 3.1](https://dotnet.microsoft.com/download)
* [Dapr CLI](https://github.com/dapr/cli)
* [Dapr DotNet SDK](https://github.com/dapr/dotnet-sdk)
* [Docker](https://www.docker.com/)
* A working [kubernetes](https://kubernetes.io/) cluster

## Create an .NET Core project and add dependencies

Open the command line and type:

```bash
dotnet new web -o dapr.k8s.secrets

cd dapr.k8s.secrets

dotnet add package Dapr.Client -v 0.5.0-preview02
```

## Update Startup.cs

Update *Startup.cs* with the following contents in order to expose a *secret* endpoint and use [Dapr](https://dapr.io) to fetch a kubernetes secret:

```csharp
namespace dapr.k8s.secrets
{
    using System.Text.Json;
    using System.Threading.Tasks;
    using Microsoft.AspNetCore.Builder;
    using Microsoft.AspNetCore.Hosting;
    using Microsoft.AspNetCore.Http;
    using Microsoft.Extensions.Configuration;
    using Microsoft.Extensions.Hosting;
    using Microsoft.Extensions.DependencyInjection;
    using Dapr.Client;
    using System;
    using System.Collections.Generic;

    public class Startup
    {
        // Dapr listens for requets on localhost
        const string localhost = "127.0.0.1";

        // Get the Dapr gRPC port
        static string daprPort => Environment.GetEnvironmentVariable("DAPR_GRPC_PORT") ?? "50001";

        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        public void ConfigureServices(IServiceCollection services)
        {
            // Create Dapr Client
            var client = new DaprClientBuilder()
                .UseEndpoint($"https://{localhost}:{daprPort}")
                .Build();

            // Add the DaprClient to DI.
            services.AddSingleton<DaprClient>(client);
        }

        public void Configure(IApplicationBuilder app, IWebHostEnvironment env, DaprClient client)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseRouting();

            app.UseEndpoints(endpoints =>
            {
                // Add the secrets route
                endpoints.MapGet("secret", Secret);
            });

            async Task Secret(HttpContext context)
            {
                // Get the secret from kubernetes
                var secretValues = await client.GetSecretAsync(
                    "kubernetes", // Name of the Dapr Secret Store
                    "super-secret", // Name of the k8s secret
                    new Dictionary<string, string>() { { "namespace", "default" } }); // Namespace where the k8s secret is deployed

                // Get the secret value
                var secretValue = secretValues["super-secret"];

                context.Response.ContentType = "application/json";
                await JsonSerializer.SerializeAsync(context.Response.Body, secretValue);
            }
        }
    }
}
```

## Create a Dockerfile with the following contents:

```csharp
FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS builder
WORKDIR /app

# caches restore result by copying csproj file separately
COPY *.csproj .
RUN dotnet restore

COPY . .
RUN dotnet publish --output /app/ --configuration Release
RUN sed -n 's:.*<AssemblyName>\(.*\)</AssemblyName>.*:\1:p' *.csproj > __assemblyname
RUN if [ ! -s __assemblyname ]; then filename=$(ls *.csproj); echo ${filename%.*} > __assemblyname; fi

# Stage 2
FROM mcr.microsoft.com/dotnet/core/aspnet:3.1
WORKDIR /app
COPY --from=builder /app .

ENV PORT 80
EXPOSE 80

ENTRYPOINT dotnet $(cat /app/__assemblyname).dll
```

## Build the Docker image and push it to a container registry

I'll be pushing the container to [Docker Hub](https://hub.docker.com/r/cmendibl3/dapr-k8s-secrets):

```shell
 docker build -t cmendibl3/dapr-k8s-secrets:1.0.0 .
 docker push cmendibl3/dapr-k8s-secrets:1.0.0
```

## Create manifest and deploy the application to kubernetes

Create a *deployment.yaml* with the following contents (remember to replace the image with your own values):

```yaml
---
# ASP.NET Core Application
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: dapr-k8s-secrets
  namespace: default
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: dapr-k8s-secrets
        aadpodidbinding: requires-vault
      annotations:
        dapr.io/enabled: "true"
        dapr.io/id: "dapr-k8s-secrets"
        dapr.io/port: "80"
    spec:
      containers:
        - name: dapr-k8s-secrets
          image: cmendibl3/dapr-k8s-secrets:1.0.0
          ports:
            - containerPort: 80
          imagePullPolicy: Always
---
# Create a Kubernetes Secret
apiVersion: v1
kind: Secret
metadata:
  name: super-secret
  namespace: default
type: Opaque
data:
  super-secret: eW91ciBzdXBlciBzZWNyZXQK

---
# If RBAC is enabled in K8s, give the default service account access to read secrets in the default namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dapr-secret-reader
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: secret-reader
subjects:
  - kind: ServiceAccount
    name: default
    namespace: default
```

If you haven't installed Dapr in your kuberntes cluster run:

```shell
dapr init --kubernetes
```

Now deploy the application to kubernetes:

```shell
kubectl apply -f ./deployment.yaml
```

## Test the application

Get the pod name and execute a port forward to test the API

```shell
$pod = kubectl get po --selector=app=dapr-k8s-secrets -n default -o jsonpath='{.items[*].metadata.name}'
kubectl port-forward $pod 80:80
```

Run the following command in other terminal:

```shell
curl http://localhost/secret
```

If everything is working you should read:

```shell
"your super secret"
```

Hope it helps!

Please find all code and files [here](https://github.com/cmendible/dotnetcore.samples/tree/main/dapr.k8s.secrets), and learn more about Dapr and the Secret API [here](https://github.com/dapr/docs/tree/master/concepts/secrets)