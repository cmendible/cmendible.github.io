---
author: Carlos Mendible
categories:
- dotNet
- dotNetCore
date: "2016-07-11T15:14:26Z"
description: 'Step by step: .NET Core and Entity Framework Core'
image: /wp-content/uploads/2016/07/efsqlite.png
tags: ["EntityFrameworkCore", "SQLite"]
title: 'Step by step: .NET Core and Entity Framework Core'
---
Today I'll show you how to create a small console application with a **Step by step: .NET Core and Entity Framework Core** example.

First be aware of the following prerequisites:

<table>
    <tr>
      <td>
        **OS**
      </td>
      <td>
        **Prerequisites**
      </td>
    </tr>
    <tr>
      <td>
        Windows
      </td>
      <td>
        Windows: You must have <a href="https://go.microsoft.com/fwlink/?LinkID=809122" target="_blank">.NET Core SDK for Windows</a> or both <a href="https://www.visualstudio.com/news/releasenotes/vs2015-update3-vs" target="_blank">Visual Studio 2015 Update 3*</a> and <a href="https://go.microsoft.com/fwlink/?LinkId=817245" target="_blank">.NET Core 1.0 for Visual Studio</a> installed.
      </td>
    </tr>
    <tr>
      <td>
        linux, mac or docker
      </td>
      <td>
        checkout <a href="https://www.microsoft.com/net/" target="_blank">.NET Core</a>
      </td>
    </tr>
</table>

Now let's start:

## 1. Create a folder for your new project
---
Open a command promt an run 
    
``` powershell
mkdir efcore
```

## 2. Create the project
---

``` powershell
cd efcore
dotnet new
```

## 3. Create a settings file
---
Create an **appsettings.json** file to hold your connection string information. We'll be using **SQLite **for this example, so add these lines:

``` json
{
  "ConnectionStrings": {
    "Sample": "Data Source=sample.db"
  }
}
```

## 4. Modify the project file
---
Modify the **project.json** to add the **EntityFrameworkCore** dependencies and also specify that the **appsettings.json** file must be copied to the output (**buildOptions **section) so it becomes available to the application once you build it.


``` json
{
  "version": "1.0.0-*",
  "buildOptions": {
    "debugType": "portable",
    "emitEntryPoint": true,
    "copyToOutput": {
      "include": "appsettings.json"
    }
  },
  "dependencies": {
    "Microsoft.Extensions.Configuration": "1.0.0",
    "Microsoft.Extensions.Configuration.Json": "1.0.0",
    "Microsoft.EntityFrameworkCore": "1.0.0",
    "Microsoft.EntityFrameworkCore.Sqlite": "1.0.0"
  },
  "frameworks": {
    "netcoreapp1.0": {
      "dependencies": {
        "Microsoft.NETCore.App": {
          "type": "platform",
          "version": "1.0.0"
        }
      },
      "imports": "dnxcore50"
    }
  }
}
```

## 5. Restore packages
---
You just modified the **project.json** file with new dependencies so please restore the packages with the following command:
    
``` powershell
dotnet restore
```

## 6. Create the Entity Framework context
---
  
Create a **SampleContext.cs** file and copy the following code 
    
``` csharp
namespace ConsoleApplication
{
    using Microsoft.EntityFrameworkCore;

    /// <summary>
    /// The entity framework context with a Students DbSet 
    /// </summary>
    public class StudentsContext : DbContext
    {
        public StudentsContext(DbContextOptions<StudentsContext> options)
            : base(options)
        { }

        public DbSet<Student> Students { get; set; }
    }

    /// <summary>
    /// A factory to create an instance of the StudentsContext 
    /// </summary>
    public static class StudentsContextFactory
    {
        public static StudentsContext Create(string connectionString)
        {
            var optionsBuilder = new DbContextOptionsBuilder<StudentsContext>();
            optionsBuilder.UseSqlite(connectionString);

            // Ensure that the SQLite database and sechema is created!
            var context = new StudentsContext(optionsBuilder.Options);
            context.Database.EnsureCreated();

            return context;
        }
    }

    /// <summary>
    /// A simple class representing a Student
    /// </summary>
    public class Student
    {
        public Student()
        {
        }

        public int Id { get; set; }

        public string Name { get; set; }

        public string LastName { get; set; }
    }
}
```

## 7. Modify Program.cs
---
Replace the contents of the **Program.cs** file with the following code 
    
``` csharp
namespace ConsoleApplication
{
    using System;
    using Microsoft.Extensions.Configuration;

    public class Program
    {
        public static void Main(string[] args)
        {
            // Enable to app to read json setting files
            var builder = new ConfigurationBuilder()
                .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true);

            var configuration = builder.Build();

            // Get the connection string
            string connectionString = configuration.GetConnectionString("Sample");

            // Create a Student instance
            var user = new Student() { Name = "Carlos", LastName = "Mendible" };

            // Add and Save the student in the database
            using (var context = StudentsContextFactory.Create(connectionString))
            {
                context.Add(user);
                context.SaveChanges();
            }

            Console.WriteLine($"Student was saved in the database with id: {user.Id}");
        }
    }
}
```

## 8. Build
---
Build the application with the following command 
    
``` powershell
dotnet build
```

## 9. Run
---

You are good to go so run the application 
    
``` powershell
dotnet run
```

You can get the code here: <https://github.com/cmendible/dotnetcore.samples/tree/master/efcore.sqlite.console>

Hope it helps!