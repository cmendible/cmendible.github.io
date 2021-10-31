---
author: Robert Bermejo
categories:
- azure
- dotnet
date: "2017-08-09T15:51:07Z"
descripton: 'Step by step: .Net Core and Azure Search'
images: ["/wp-content/uploads/2017/08/Azure-Search.png"]
tags: ["azure search"]
title: 'Step by step: .Net Core and Azure Search'
url: /2017/08/09/step-by-step-net-core-and-azure-search/
---
**Step by step: .Net Core and Azure Search** is small introduction on how to connect to <a href="https://azure.microsoft.com/en-us/services/search/" target="_blank">Azure Search</a>, create and delete indexes, models, add documents and perform basic queries.

Let's go for it:

## 1. Create an Azure Search service
---
Create an <a href="https://azure.microsoft.com/en-us/services/search/" target="_blank">Azure Search</a> service in your Azure subscription, and get the Azure Search name and primary Read-Write key.
    
Find the steps <a href="https://docs.microsoft.com/en-us/azure/search/search-create-service-portal" target="_blank">here</a>.
      
## 2. Create a folder for your project
---
Open a command prompt and run:
          
``` powershell
mkdir netCoreAzureSearch
```

## 3. Create the project
``` powershell
cd netCoreAzureSearch
dotnet new console
```

## 4. Add a reference to Microsoft.Azure.Search
---
<a href="https://www.nuget.org/packages/Microsoft.Azure.Search" target="_blank">Microsoft.Azure.Search</a> will allow your application to talk with Azure Search.
          
``` powershell
dotnet add package  Microsoft.Azure.Search -v 3.0.4
dotnet restore
```

## 5. Replace the content of Program.cs
---
Replace the content of Program.cs with the following contents and be sure to replace the values in lines 13 and 16 with the Azure Search service name and key from step 1:
          
``` csharp
namespace netCoreAzureSearch
{
    using System;
    using Microsoft.Azure.Search;
    using System.Linq;
    using System.Collections.Generic;
    using Microsoft.Azure.Search.Models;
    
    class Program
    {
        static void Main(string[] args)
        {
            var azureSearchName = "[Your Azure Search Name]";
 
            // The key to you cosmosdb
            var azureSearchKey = "[Your Azure Search Key]";

            // Index name variable
            string indexName = "books";
            
            // Create a service client connection
            ISearchServiceClient azureSearchService = new SearchServiceClient(azureSearchName, new SearchCredentials(azureSearchKey));
             
            // Get the Azure Search Index
            ISearchIndexClient indexClient = azureSearchService.Indexes.GetClient(indexName);

            // If the Azure Search Index exists delete it.
            Console.WriteLine( "Deleting index...\n"); 
            if (azureSearchService.Indexes.Exists(indexName))
            {
                 azureSearchService.Indexes.Delete(indexName);
            }

            // Create an Index Model
            Console.WriteLine( "Creating index Model...\n"); 
            Index indexModel = new Index()
            {
                Name = indexName,
                Fields = new[]
                {
                    new Field("ISBN", DataType.String) { IsKey = true, IsRetrievable = true, IsFacetable = false },
                    new Field("Titulo", DataType.String) {IsRetrievable = true, IsSearchable = true, IsFacetable = false },
                    new Field("Autores", DataType.Collection(DataType.String)) {IsSearchable = true, IsRetrievable = true, IsFilterable = true, IsFacetable = false },
                    new Field("FechaPublicacion", DataType.DateTimeOffset) { IsFilterable = true, IsRetrievable = false, IsSortable = true, IsFacetable = false },
                    new Field("Categoria", DataType.String) { IsFacetable = true, IsFilterable= true, IsRetrievable = true }

                }                   
            };
            
            // Create the Index in AzureSearch
            Console.WriteLine( "Creating index...\n"); 
            var resultIndex = azureSearchService.Indexes.Create(indexModel);

            // Add documents to the Index
            Console.WriteLine( "Add documents...\n");
            var listBooks = new BookModel().GetBooks();
            indexClient.Documents.Index(IndexBatch.MergeOrUpload<BookModel>(listBooks));
            System.Threading.Thread.Sleep(1000);

            // Search by word
            Console.WriteLine("{0}", "Searching documents 'Cloud'...\n");
            Search(indexClient, searchText: "Cloud");
            System.Threading.Thread.Sleep(1000);

            // Search everything and filter
            Console.WriteLine("\n{0}", "Filter documents by Autores 'Eugenio Betts'...\n");
            Search(indexClient, searchText: "*", filter: "Autores / any(t: t eq 'Eugenio Betts')");
            System.Threading.Thread.Sleep(1000);

            // Search everything and order
            Console.WriteLine("\n{0}", "order by FechaPublicacion\n");
            Search(indexClient, searchText: "*", order: new List<string>() { "FechaPublicacion" });
            System.Threading.Thread.Sleep(1000);

            //Search everything and facet
            Console.WriteLine("\n{0}", "Facet by Categoria \n");
            Search(indexClient, searchText: "*", facets: new List<string>() { "Categoria" });
            System.Threading.Thread.Sleep(1000);
        }

         private static void Search(ISearchIndexClient indexClient, string searchText, string filter = null, List<string> order = null, List<string> facets = null)
        {
            // Execute search based on search text and optional filter
            var sp = new SearchParameters();

            // Add Filter
            if (!String.IsNullOrEmpty(filter))
            {
                sp.Filter = filter;
            }

            // Order
            if (order != null && order.Count > 0)
            {
                sp.OrderBy = order;
            }

            // facets
            if (facets != null && facets.Count > 0)
            {
                sp.Facets = facets;
            }

            // Search
            DocumentSearchResult<BookModel> response = indexClient.Documents.Search<BookModel>(searchText, sp);
           
            foreach (SearchResult<BookModel> result in response.Results)
            {
                Console.WriteLine(result.Document + " - Score: " + result.Score);
            }

            if (response.Facets != null)
            {
                foreach (var facet in response.Facets)
                {
                    Console.WriteLine("\n Facet Name: " + facet.Key);
                    foreach (var value in facet.Value)
                    {
                        Console.WriteLine("Value :" + value.Value + " - Count: " + value.Count);
                    }
                }
            }
        }
    }

    /// <summary>
    /// A simple class representing a book
    /// </summary>
    public  class BookModel
    {
        public string ISBN { get; set; }
        public string Titulo { get; set; }
        public List<string> Autores { get; set; }
        public DateTimeOffset FechaPublicacion { get; set; }
        public string Categoria { get; set; }

        public List<BookModel> GetBooks()
        {
            var listBooks = new List<BookModel>();
            listBooks.Add(new BookModel()
            {
                ISBN = "9781430224792",
                Titulo = "Windows Azure Platform (Expert's Voice in .NET)",
                Categoria = "Comic",
                Autores = new List<string>() {"Redkar", "Tejasw" },
                FechaPublicacion = DateTimeOffset.Now.AddDays(-2)
            });

            listBooks.Add(new BookModel()
            {
                ISBN = "9780470506387",
                Titulo = "Cloud Computing with the Windows Azure Platform",
                Categoria = "Terror",
                Autores = new List<string>() { "Roger Jennings"},
                FechaPublicacion = DateTimeOffset.Now
            });

            listBooks.Add(new BookModel()
            {
                ISBN = "9780889222861",
                Titulo = "Azure Blues",
                Categoria = "Terror",
                Autores = new List<string>() { "Gilbert", "Gerry", "Rogery Landing"},                
                FechaPublicacion = DateTimeOffset.Now.AddMonths(-3)
            });

            listBooks.Add(new BookModel()
            {
                ISBN = "9780735649675",
                Titulo = "Moving Applications to the Cloud on the Microsoft Azure(TM) Platform",
                Categoria = "Fiction",
                Autores = new List<string>() { "Pace", "Eugenio Betts", "Dominic Densmore", "Scott; Dunn", "Ryan Narumoto", "Masashi Woloski", "Matias" },
                FechaPublicacion = DateTimeOffset.Now
            });

            return listBooks;
        }

        public override string ToString()
        {
            return "ISBN: " + this.ISBN + " - titulo: " + this.Titulo + " - Autores: " + String.Join(", ", this.Autores);     
        }
    }
}
```

## 6. Run the application
---
Run the following commands:
          
``` powershell
dotnet build
dotnet run
```
 
You can get the code <a href="https://github.com/bermejoblasco/NetCoreAzureSearchSample" target="_blank">here</a>.

See you soon!