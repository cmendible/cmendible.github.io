---
author: Carlos Mendible
categories:
- dotnet
crosspost_to_medium: true
date: "2017-04-29T19:51:45Z"
description: 'Step by step: Couchbase Lite with .Net Core'
images: ["/wp-content/uploads/2017/04/Couchbase_Inc._official_logo.png"]
tags: ["couchbase", "Docker", "NoSQL"]
title: 'Step by step: Couchbase Lite with .Net Core'
url: /2017/04/29/step-by-step-couchbase-lite-with-net-core/
---
Not long after writing **Step by step: Couchbase with .Net Core** I discovered Couchbase Lite, which is still in development, but it looks like a great solution for embedded NoSQL scenarios.

So let's start with this simple: **Step by step: Couchbase Lite with .Net Core** tutorial.

## 1. Create a folder for your new project
---
Open a command prompt an run 
    
``` powershell
mkdir couchbase.lite.console
cd couchbase.lite.console
```

## 2. Create a console project
---

``` powershell
dotnet new console
```

## 3. Add a nuget.config
---
Since Couchbase Lite is still in development you'll need to add their development nuget server as Nuget source, so create a nuget.config file with the following contents.
    
``` xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="Couchbase.Developer" value="http://mobile.nuget.couchbase.com/nuget/Developer/" />
  </packageSources>
  <disabledPackageSources />
</configuration>
```

## 4. Add a reference to the Couchbase Lite nuget package
---
Run the following command 
    
``` powershell
dotnet add package couchbase.lite -v 2.0.0-db004
dotnet restore
```

## 5. Replace the contents of Program.cs
---
Replace the contents of the **Program.cs** file with the following code:

    
``` csharp
using System;
using System.Collections.Generic;
using System.IO;
using Couchbase.Lite;
using Couchbase.Lite.Query;

namespace couchbase.lite
{
    class Program
    {
        static void Main(string[] args)
        {
            // Get current directory to store the database.
            var dir = Directory.GetCurrentDirectory();

            // Delete the database so we can run the sample without issues.
            DatabaseFactory.DeleteDatabase("db", dir);

            // Create the default options.
            var options = DatabaseOptions.Default;
            options.Directory = dir;

            // Create the database
            var db = DatabaseFactory.Create("db", options);

            // Create a document the Id == "Polar Ice" an set properties. 
            var document = db.GetDocument("Polar Ice");
            document.Properties = new Dictionary<string, object>
            {
                ["name"] = "Polar Ice",
                ["brewery_id"] = "Polar"
            };

            // Save the document
            document.Save();

            // Query for the document abd write results to the console.
            var query = QueryFactory.Select()
                .From(DataSourceFactory.Database(db))
                .Where(ExpressionFactory.Property("brewery_id").EqualTo("Polar"));

            var rows = query.Run();
            foreach (var row in rows)
            {
                Console.WriteLine($"Fetched doc with id :: {row.DocumentID}");
            }
        }
    }
}
```

## 6. Run the program
---
Run the program 
    
``` powershell
dotnet run
```
The console should show the following message:

    
``` powershell
Fetched doc with id :: Polar Ice
```

Get the code here: <a href="https://github.com/cmendible/dotnetcore.samples/tree/master/couchbase.lite.console"  target="_blank">https://github.com/cmendible/dotnetcore.samples/tree/master/couchbase.lite.console</a>

Hope it helps!