---
author: Carlos Mendible
categories:
- azure
- dotnet
date: "2017-06-25T18:52:22Z"
description: 'Step by step: .NET Core and Azure Cosmos DB'
images: ["/wp-content/uploads/2017/06/cosmos.png"]
tags: ["cosmos db"]
title: 'Step by step: .NET Core and Azure Cosmos DB'
url: /2017/06/25/step-by-step-net-core-and-azure-cosmos-db/
---
**Step by step: .NET Core and Azure Cosmos DB** is a short post on how to connect to Cosmos DB, save a document and then query to find it.

Let's start:

## Create a Cosmos DB account
---
Create a Cosmos DB account in your Azure subscription. Once created get the URI and the primary Read-write key from the Keys section.

If you need info on how to do this browse to the **Create a database account** section here: <a href="https://docs.microsoft.com/en-us/azure/cosmos-db/create-documentdb-dotnet" target="_blank">https://docs.microsoft.com/en-us/azure/cosmos-db/create-documentdb-dotnet</a>
      
## Create a folder for your new project
---
Open a command promt an run 
          
``` powershell
mkdir cosmosdb
```
      
## Create the project
---      
      
``` powershell
cd cosmosdb
dotnet new console
```

## Add a reference to the Microsoft.Azure.DocumentDB.Core library
---      
Add a reference to the Microsoft.Azure.DocumentDB.Core client library so you are able to talk with Cosmos Db.
          
``` powershell
dotnet add package Microsoft.Azure.DocumentDB.Core -v 1.3.2
```    
      
## Restore packages
---
You just modified the project dependencies so please restore the packages with the following command:
     
          
``` powershell
dotnet restore
```
      
## Modify Program.cs
---
Replace the contents of the **Program.cs** file with the following code and remember to change lines 13 and 16 with your account URI and Key 
          
``` csharp
using System;
using System.Linq;
using Microsoft.Azure.Documents;
using Microsoft.Azure.Documents.Client;

namespace cosmosdb
{
    class Program
    {
        static void Main(string[] args)
        {
            // The endpoint to your cosmosdb instance
            var endpointUrl = "[THE ENPOINT OF YOUR COSMOSDB SERVICE HERE]";

            // The key to you cosmosdb
            var key = "[THE KEY OF YOUR COSMOSDB SERVICE HERE]";

            // The name of the database
            var databaseName = "Students";

            // The name of the collection of json documents
            var databaseCollection = "StudentsCollection";

            // Create a cosmosdb client
            using (var client = new DocumentClient(new Uri(endpointUrl), key))
            {
                // Create the database
                client.CreateDatabaseIfNotExistsAsync(new Database() { Id = databaseName }).GetAwaiter().GetResult();

                // Create the collection
                client.CreateDocumentCollectionIfNotExistsAsync(
                    UriFactory.CreateDatabaseUri(databaseName),
                    new DocumentCollection { Id = databaseCollection }).
                    GetAwaiter()
                    .GetResult();

                // Create a Student instance
                var student = new Student() { Id = "Student.1", Name = "Carlos", LastName = "Mendible" };

                // Sava the document to cosmosdb
                client.CreateDocumentAsync(UriFactory.CreateDocumentCollectionUri(databaseName, databaseCollection), student)
                    .GetAwaiter().GetResult();

                Console.WriteLine($"Student was saved in the database with id: {student.Id}");

                // Query for the student by last name
                var query = client.CreateDocumentQuery<Student>(
                        UriFactory.CreateDocumentCollectionUri(databaseName, databaseCollection))
                        .Where(f => f.LastName == "Mendible")
                        .ToList();

                if (query.Any())
                {
                    Console.WriteLine("Student was found in the cosmosdb database");
                }

            }
        }
    }

    /// <summary>
    /// A simple class representing a Student
    /// </summary>
    public class Student
    {
        public string Id { get; set; }

        public string Name { get; set; }

        public string LastName { get; set; }
    }
}
```
      
## Build
---      

Build the application with the following command 
          
``` powershell
dotnet build
```
      
## Run
---
You are good to go so run the application 
          
``` powershell
dotnet run
```

You can get the code here: <a href="https://github.com/cmendible/dotnetcore.samples/tree/main/cosmosdb">https://github.com/cmendible/dotnetcore.samples/tree/main/cosmosdb</a>
  
Hope it helps!  