---
author: Carlos Mendible
categories:
- dotnet
crosspost_to_medium: true
date: "2017-04-10T17:11:15Z"
description: 'Step by step: Couchbase with .Net Core'
images: ["/wp-content/uploads/2017/04/Couchbase_Inc._official_logo.png"]
tags: ["couchbase", "Docker", "NoSQL"]
title: 'Step by step: Couchbase with .Net Core'
url: /2017/04/10/step-by-step-couchbase-with-net-core/
---
This week I started to read an understand how **[Couchbase](http://www.couchbase.com)** works and that's the reason I decided to write: **Step by step: Couchbase with .Net Core**

**Tip:** I'll be using **Docker** to install and run **[Couchbase](http://www.couchbase.com)**

Now let's start:

## 1. Create a folder for your new project
---
Open a command prompt an run 
    
``` powershell
mkdir couchbase.console
cd couchbase.console
```

## 2. Create a console project
--- 

``` powershell
dotnet new console
```
## 3. Add the Couchbase nuget package
---
Add the **<a href="http://www.couchbase.com">Couchbase</a>** nuget package to your project:
    
``` powershell
dotnet add package CouchbaseNetClient
dotnet restore
```

## 4. Replace the contents of Program.cs
---
Replace the contents of the **Program.cs** file with the following code:
    
``` csharp
namespace couchbase.console
{
    using System;
    using Couchbase;

    class Program
    {
        static void Main(string[] args)
        {
            // Connect to cluster. Defaults to localhost
            using (var cluster = new Cluster())
            {
                // Open the beer sample bucket
                using (var bucket = cluster.OpenBucket("beer-sample"))
                {
                    // Create a new beer document
                    var document = new Document<dynamic>
                    {
                        Id = "Polar Ice",
                        Content = new
                        {
                            name = "Polar Ice",
                            brewery_id = "Polar"
                        }
                    };

                    // Insert the beer document
                    var result = bucket.Insert(document);
                    if (result.Success)
                    {
                        Console.WriteLine("Inserted document '{0}'", document.Id);
                    }

                    // Query the beer sample bucket and find the beer we just added.
                    using (var queryResult = bucket.Query<dynamic>("SELECT name FROM `beer-sample` WHERE brewery_id =\"Polar\""))
                    {
                        foreach (var row in queryResult.Rows)
                        {
                            Console.WriteLine(row);
                        }
                    }
                }
            }
        }
    }
}
```

## 5. Setup Couchbase with Docker
---
Run the following commands:
    
``` powershell
docker pull couchbase/server
docker run -d --name db -p 8091-8094:8091-8094 -p 11210:11210 couchbase
```
    
Browse to: <a href="http://localhost:8091" target="_blank">http://localhost:8091</a> and setup **Couchbase**.

Be sure to add the Beer Sample bucket and check the documentation here: <a href="https://hub.docker.com/r/couchbase/server/" target="_blank">https://hub.docker.com/r/couchbase/server/</a>

## 6. Run the program
---      
Run the program and enjoy!
          
``` powershell
dotnet run
```

Get the code here: <a href="https://github.com/cmendible/dotnetcore.samples/tree/master/couchbase.console"  target="_blank">https://github.com/cmendible/dotnetcore.samples/tree/master/couchbase.console</a>
  
Hope it helps!  