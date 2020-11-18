---
author: Carlos Mendible
categories:
- dotnet
crosspost_to_medium: true
date: "2020-11-18T10:00:00Z"
description: 'Dapr: Reading local secrets with .NET 5'
images: ["/assets/img/posts/dapr.png"]
published: true
tags: ["dapr"]
title: 'Dapr: Reading local secrets with .NET 5'
---

Now that [Dapr](https://dapr.io/) is about to hit version 1.0.0 let me show you how easy is to read secrets with a [.NET 5](https://dotnet.microsoft.com/?WT.mc_id=AZ-MVP-5002618) console application.

## Create a console application

``` shell
dotnet new console -n DaprSecretSample
cd DaprSecretSample
```

## Add a reference to the Dapr.Client library

``` shell
dotnet add package Dapr.Client --prerelease   
```

## Create a Secret Store component

Create a `components` folder and inside place a file named `secretstore.yaml` with the following contents:

``` yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: starwarssecrets
spec:
  type: secretstores.local.file
  metadata:
    - name: secretsFile
      value: ./secrets.json
```

> This component will enable Dapr, and therefore your application, to read secrets from a `secrets.json` file.

## Create a Secrets file

Create a `secrets.json` file with the following contents:

``` json
{
    "mandalorianSecret": "this is the way!"
}
```

## Replace the contents of Program.cs

Replace the contents of `Program.cs` with the following code:

``` csharp
using System;
using Dapr.Client;

var client = new DaprClientBuilder().Build();
var secret = await client.GetSecretAsync("starwarssecrets", "mandalorianSecret");
Console.WriteLine($"Secret from local file: {secret["mandalorianSecret"]});
```

## Test the program

Run the following command to test the application.

``` shell
dapr run --app-id secretapp --components-path .\components\ -- dotnet run
```

## Experiment

>  One of the amazing things about Dapr is that your code will be the same even if you change the underlying secret store.

In your local environment you can also try reading "secrets" from environment variables. In order to do so, replace the contents of the `./componentes/secrets.yaml` file with:

``` yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: starwarssecrets
spec:
  type: secretstores.local.env
```

Be sure to set, in your system, an environment variable named `mandalorianSecret`, for instance:

``` shell
export mandalorianSecret="May The Force be with you"
```

and run the application again:

``` shell
dapr run --app-id secretapp --components-path .\components\ -- dotnet run
```

> **Note:** I recommend using the secret stores shown in this post only for development or test scenarios. For a complete list of supported secret stores check the following repo: https://github.com/dapr/components-contrib/tree/master/secretstores.

Hope it helps!