---
author: Carlos Mendible
categories:
- dotNet
date: "2016-06-12T20:24:54Z"
description: Remove RavenDb's API from Swagger
image: /wp-content/uploads/2016/06/swagger.png
tags: ["ASP.Net", "OWIN", "RavenDb", "Swagger", "Swashbuckle", "WebAPI"]
title: Remove RavenDb's API from Swagger
---
I've been working on a small IoT project where we expose a simple REST API and use <a href="https://ravendb.net/" target="_blank">RavenDB </a>Embedded as the database. Once we enabled <a href="http://swagger.io/" target="_blank">Swagger</a> on our project we noticed that the generated documentation was not only showing our API but those exposed by RavenDB too. So the question came up: how can we **Remove RavenDb's API from <a href="http://swagger.io/" target="_blank">Swagger</a>** Documentation?

Turns out that **<a href="https://github.com/domaindrivendev/Swashbuckle" target="_blank">Swashbuckle.Swagger</a>** let's you create your own Document Filters implementing the **IDocumentFilter** interface.

Below you'll find the fast solution I came up with, and it's easy to modify if you need to remove any other type from your Swagger documentation.

``` csharp
    using Swashbuckle.Swagger;
    using System.Collections.Generic;
    using System.Linq;
    using System.Web.Http.Description;

    public class HideRavenDocumentFilter : IDocumentFilter
    {
        public void Apply(SwaggerDocument swaggerDoc, SchemaRegistry schemaRegistry, IApiExplorer apiExplorer)
        {
            foreach (var apiDescription in apiExplorer.ApiDescriptions)
            {
                if (!apiDescription.ActionDescriptor.ControllerDescriptor.ControllerType.Namespace.StartsWith("Raven"))
                {
                    continue;
                }

                var routeToRemove = swaggerDoc.paths.Where(r => r.Key.Contains(apiDescription.ActionDescriptor.ControllerDescriptor.ControllerName))
                    .FirstOrDefault();

                if (!routeToRemove.Equals(default(KeyValuePair<string, PathItem>)))
                {
                    swaggerDoc.paths.Remove(routeToRemove);
                }
            }
        }
    }
```

Hope it helps!