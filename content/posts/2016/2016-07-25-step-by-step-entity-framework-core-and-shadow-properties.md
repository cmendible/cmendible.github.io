---
author: Carlos Mendible
categories:
- dotnet
date: "2016-07-25T16:07:49Z"
description: 'Step by step: Entity Framework Core and Shadow Properties'
images: ["/wp-content/uploads/2016/07/efsqlite.png"]
tags: ["EntityFrameworkCore"]
title: 'Step by step: Entity Framework Core and Shadow Properties'
---
Today I'll show you how to create a small console application with a **Step by step: Entity Framework Core and Shadow Properties** example.

We'll be modifying the code from my previous post: Step by step: <a href="https://carlos.mendible.com/2016/07/11/step-by-step-dotnet-core-and-entity-framework-core/" target="_blank">.NET Core and Entity Framework Core</a> to make the Student entity auditable without creating public/protected properties or private fields to hold the typical audit info (CreatedBy, CreatedAt, UpdatedBy and UpdatedAt) database fields.

This means that your domain entities won't expose the audit information which almost in every case is not part of the business behavior you'll be trying to model.

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
mkdir efcore.shadowproperties.console
```

## 2. Create the project
---

``` powershell
cd efcore.shadowproperties.console
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
  
Modify the **project.json** to add the **EntityFrameworkCore** dependencies and also specify that the **appsettings.json** file must be copied to the output (**buildOptions**section) so it becomes available to the application once you build it.

Note that you'll also have to add **System.Security.Claims** to simulate an authenticated user:
    
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
    "Microsoft.EntityFrameworkCore.Sqlite": "1.0.0",
    "System.Security.Claims": "4.0.1"
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

## 6. Create a marker interface in order to make your entities auditable
---
Every entity you mark with this interface will save audit info to the database.

``` csharp
namespace ConsoleApplication
{
    public interface IAuditable
    {

    }   
}
```

## 7. Create the Entity Framework context
---
Create a **SampleContext.cs** file and copy the following code 
    
``` csharp
namespace ConsoleApplication
{
    using System;
    using System.Linq;
    using System.Reflection;
    using Microsoft.EntityFrameworkCore;
    using Microsoft.EntityFrameworkCore.ChangeTracking;
    using System.Security.Claims;

    /// <summary>
    /// The entity framework context with a Students DbSet 
    /// </summary>
    public class StudentsContext : DbContext
    {
        public StudentsContext(DbContextOptions<StudentsContext> options)
            : base(options)
        { }

        public DbSet<Student> Students { get; set; }

        /// <summary>
        /// We overrride OnModelCreating to map the audit properties for every entity marked with the 
        /// IAuditable interface.
        /// </summary>
        /// 
        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            foreach (var entityType in modelBuilder.Model.GetEntityTypes()
                .Where(e => typeof(IAuditable).IsAssignableFrom(e.ClrType)))
            {
                    modelBuilder.Entity(entityType.ClrType)
                        .Property<DateTime>("CreatedAt");

                    modelBuilder.Entity(entityType.ClrType)
                        .Property<DateTime>("UpdatedAt");

                    modelBuilder.Entity(entityType.ClrType)
                        .Property<string>("CreatedBy");

                    modelBuilder.Entity(entityType.ClrType)
                        .Property<string>("UpdatedBy");
            }

            base.OnModelCreating(modelBuilder);
        }

        /// <summary>
        /// Override SaveChanges so we can call the new AuditEntities method.
        /// </summary>
        /// <returns></returns>
        public override int SaveChanges()
        {
            this.AuditEntities();

            return base.SaveChanges();
        }
        
        /// <summary>
        /// Method that will set the Audit properties for every added or modified Entity marked with the 
        /// IAuditable interface.
        /// </summary>
        private void AuditEntities()
        {

            // Get the authenticated user name 
            string userName = string.Empty;

            var user = ClaimsPrincipal.Current;
            if (user != null)
            {
                var identity = user.Identity;
                if (identity != null)
                {
                    userName = identity.Name;
                }
            }

            // Get current date & time
            DateTime now = DateTime.Now;

            // For every changed entity marked as IAditable set the values for the audit properties
            foreach (EntityEntry<IAuditable> entry in ChangeTracker.Entries<IAuditable>())
            {
                // If the entity was added.
                if (entry.State == EntityState.Added)
                {
                    entry.Property("CreatedBy").CurrentValue = userName;
                    entry.Property("CreatedAt").CurrentValue = now;
                }
                else if (entry.State == EntityState.Modified) // If the entity was updated
                {
                    entry.Property("UpdatedBy").CurrentValue = userName;
                    entry.Property("UpdatedAt").CurrentValue = now;
                }
            }
        }
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

            var context = new StudentsContext(optionsBuilder.Options);
            context.Database.EnsureCreated();

            return context;
        }
    }

    /// <summary>
    /// A simple class representing a Student.
    /// We've marked this entity as IAditable so we can save audit info for the entity.
    /// </summary>
    public class Student : IAuditable
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

Note that the beauty of shadow properties is that both the IAuditable interface and the Student entity don't expose Audit info. We are mapping properties that are not represented in our domain model!

## 8. Modify Program.cs
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

            // With this code we are going to simulate an authenticated user.
            ClaimsPrincipal.ClaimsPrincipalSelector = () =>
                {
                    return new ClaimsPrincipal(new ClaimsIdentity(new GenericIdentity("cmendibl3")));
                };

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

Line 22 is the one were we simulate an authenticated user.

## 9. Build
---
Build the application with the following command 
    
``` powershell
dotnet build
```

## 10. Run
---
You are good to go so run the application 
    
``` powershell
dotnet run
```
    
If you check your SQlite database with a tool such as <a href="http://sqlitebrowser.org/" target="_blank">DB Browser for SQlite</a> you'll see the four Audit properties in the Students table as shown in the following picture:

<a href="/wp-content/uploads/2016/07/students.png"><img src="/wp-content/uploads/2016/07/students.png" alt="students" class="alignleft wp-image-5391" srcset="/wp-content/uploads/2016/07/students.png 1455w, /wp-content/uploads/2016/07/students-300x55.png 300w, /wp-content/uploads/2016/07/students-768x141.png 768w, /wp-content/uploads/2016/07/students-1024x188.png 1024w, /wp-content/uploads/2016/07/students-250x46.png 250w" sizes="(max-width: 1455px) 100vw, 1455px" /></a>

You can get the code here: <https://github.com/cmendible/dotnetcore.samples/tree/main/efcore.shadowproperties.console>

Hope it helps!