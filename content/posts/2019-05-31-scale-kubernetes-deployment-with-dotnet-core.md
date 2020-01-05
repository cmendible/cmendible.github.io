---
author: Carlos Mendible
categories:
- dotnetcore
- kubernetes
- devops
crosspost_to_medium: true
date: "2019-05-31T09:12:00Z"
description: Scale a Kubernetes Deployment with .NET Core
image: /assets/img/posts/kubernetes.png
published: true
# tags: aspNetCore
title: Scale a Kubernetes Deployment with .NET Core
---

Let's start:

## 1. Create a folder for your new project
---

Open a command prompt an run:

``` shell
mkdir kuberenetes.scale
```

## 2. Create the project
---

``` shell
cd kuberenetes.scale
dotnet new api
```

## 3. Add the references to KubernetesClient
---

``` shell
dotnet add package KubernetesClient -v 1.5.18
dotnet restore
```

## 4. Create a PodsController.cs with the following code
---

``` csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using k8s;
using k8s.Models;
using Microsoft.AspNetCore.JsonPatch;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Configuration;

namespace kubernetes.scale
{
    [Route("api/[controller]")]
    [ApiController]
    public class PodsController : ControllerBase
    {
        private KubernetesClientConfiguration k8sConfig = null;

        public PodsController(IConfiguration config)
        {
            // Reading configuration to know if running inside a cluster or in local mode.
            var useKubeConfig = bool.Parse(config["UseKubeConfig"]);
            if (!useKubeConfig)
            {
                // Running inside a k8s cluser
                k8sConfig = KubernetesClientConfiguration.InClusterConfig();
            }
            else
            {
                // Running on dev machine
                k8sConfig = KubernetesClientConfiguration.BuildConfigFromConfigFile();
            }
        }

        [HttpPatch("scale")]
        public IActionResult Scale([FromBody]ReplicaRequest request)
        {
            // Use the config object to create a client.
            using (var client = new Kubernetes(k8sConfig))
            {
                // Create a json patch for the replicas
                var jsonPatch = new JsonPatchDocument<V1Scale>();
                // Set the new number of repplcias
                jsonPatch.Replace(e => e.Spec.Replicas, request.Replicas);
                // Creat the patch
                var patch = new V1Patch(jsonPatch);

                // Patch the "minions" Deployment in the "default" namespace
                client.PatchNamespacedDeploymentScale(patch, request.Deployment, request.Namespace);

                return NoContent();
            }
        }
    }

    public class ReplicaRequest
    {
        public string Deployment { get; set; }
        public string Namespace { get; set; }
        public int Replicas { get; set; }
    }
}
```

## 5. Replace the contents of the appsettings.Development.json file

Note the **UseKubeConfig** property is set to **true**.

``` json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "System": "Information",
      "Microsoft": "Information"
    }
  },
  "UseKubeConfig": true
}
```

## 6. Replace the contents of the appsettings.json file

Note the **UseKubeConfig** property is set to **false**.

``` json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  },
  "AllowedHosts": "*",
  "UseKubeConfig": false
}
```

## 7. Test the application
---

Test the app from your dev machine running:

``` shell
dotnet run
curl -k -i -H 'Content-Type: application/json' -d '{"Deployment": "<YOUR DEPLOYMENT NAME>", "Namespace": "<YOUR NAMESPACE>", "Replicas": 3}' -X PATCH https://localhost:5001/api/pods/scale
```

**Note**: You must have a valid config file to connect to the k8s cluster.

Please find all the code used in this post [here](https://github.com/cmendible/dotnetcore.samples/tree/master/kubernetes.scale).

Hope it helps!
