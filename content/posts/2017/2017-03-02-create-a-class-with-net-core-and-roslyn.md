---
author: Carlos Mendible
categories:
- dotnet
crosspost_to_medium: true
date: "2017-03-02T07:28:14Z"
description: Create a class with .NET Core and Roslyn
images: ["/wp-content/uploads/2017/01/roslyn.png"]
tags: ["Roslyn"]
title: Create a class with .NET Core and Roslyn
url: /2017/03/02/create-a-class-with-net-core-and-roslyn/
---
After my post first post on: **<a href="https://carlos.mendible.com/2017/01/29/net-core-roslyn-and-code-generation/" target="_blank">Code Generation</a>** I decided to go a bit further, so today we'll **Create a class with .NET Core and Roslyn** and write the output to the console.

Let's see:

## 1. Create the application
---
Open a command prompt and run 

``` powershell
    md roslyn.create.class
    cd roslyn.create.class
    dotnet new
    dotnet restore
    code .
```

## 2. Replace the contents of project.json
---
Replace the contents on **project.json** file in order to include the references to: **Microsoft.CodeAnalysis.CSharp** and **Microsoft.CodeAnalysis.Compilers<br /> **
    
``` json
{
  "version": "1.0.0-*",
  "buildOptions": {
    "debugType": "portable",
    "emitEntryPoint": true
  },
  "dependencies": {
    "Microsoft.CodeAnalysis.CSharp": "2.0.0-rc4",
    "Microsoft.CodeAnalysis.Compilers": "2.0.0-rc4"
  },
  "frameworks": {
    "netcoreapp1.1": {
      "dependencies": {
        "Microsoft.NETCore.App": {
          "type": "platform",
          "version": "1.1.0"
        }
      },
      "imports": "dnxcore50"
    }
  }
}
```

## 3. Replace the contents of Program.cs
---
The **CreateClass** method is where the magic occurs. Each relevant line is explained to help you understand each step.
    
``` csharp
using System;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;

namespace Roslyn.CodeGeneration
{
    public class Program
    {
        public static void Main(string[] args)
        {
            // Create a class
            CreateClass();

            // Wait to exit.
            Console.Read();
        }

        /// <summary>
        /// Create a class from scratch.
        /// </summary>
        static void CreateClass()
        {
            // Create a namespace: (namespace CodeGenerationSample)
            var @namespace = SyntaxFactory.NamespaceDeclaration(SyntaxFactory.ParseName("CodeGenerationSample")).NormalizeWhitespace();

             // Add System using statement: (using System)
            @namespace = @namespace.AddUsings(SyntaxFactory.UsingDirective(SyntaxFactory.ParseName("System")));

            //  Create a class: (class Order)
            var classDeclaration = SyntaxFactory.ClassDeclaration("Order");

            // Add the public modifier: (public class Order)
            classDeclaration = classDeclaration.AddModifiers(SyntaxFactory.Token(SyntaxKind.PublicKeyword));

            // Inherit BaseEntity<T> and implement IHaveIdentity: (public class Order : BaseEntity<T>, IHaveIdentity)
            classDeclaration = classDeclaration.AddBaseListTypes(
                SyntaxFactory.SimpleBaseType(SyntaxFactory.ParseTypeName("BaseEntity<Order>")),
                SyntaxFactory.SimpleBaseType(SyntaxFactory.ParseTypeName("IHaveIdentity")));

            // Create a string variable: (bool canceled;)
            var variableDeclaration = SyntaxFactory.VariableDeclaration(SyntaxFactory.ParseTypeName("bool"))
                .AddVariables(SyntaxFactory.VariableDeclarator("canceled"));

            // Create a field declaration: (private bool canceled;)
            var fieldDeclaration = SyntaxFactory.FieldDeclaration(variableDeclaration)
                .AddModifiers(SyntaxFactory.Token(SyntaxKind.PrivateKeyword));

            // Create a Property: (public int Quantity { get; set; })
            var propertyDeclaration = SyntaxFactory.PropertyDeclaration(SyntaxFactory.ParseTypeName("int"), "Quantity")
                .AddModifiers(SyntaxFactory.Token(SyntaxKind.PublicKeyword))
                .AddAccessorListAccessors(
                    SyntaxFactory.AccessorDeclaration(SyntaxKind.GetAccessorDeclaration).WithSemicolonToken(SyntaxFactory.Token(SyntaxKind.SemicolonToken)),
                    SyntaxFactory.AccessorDeclaration(SyntaxKind.SetAccessorDeclaration).WithSemicolonToken(SyntaxFactory.Token(SyntaxKind.SemicolonToken)));

            // Create a stament with the body of a method.
            var syntax = SyntaxFactory.ParseStatement("canceled = true;");

            // Create a method
            var methodDeclaration = SyntaxFactory.MethodDeclaration(SyntaxFactory.ParseTypeName("void"), "MarkAsCanceled")
                .AddModifiers(SyntaxFactory.Token(SyntaxKind.PublicKeyword))
                .WithBody(SyntaxFactory.Block(syntax));

            // Add the field, the property and method to the class.
            classDeclaration = classDeclaration.AddMembers(fieldDeclaration, propertyDeclaration, methodDeclaration);

            // Add the class to the namespace.
            @namespace = @namespace.AddMembers(classDeclaration);

            // Normalize and get code as string.
            var code = @namespace
                .NormalizeWhitespace()
                .ToFullString();

            // Output new code to the console.
            Console.WriteLine(code);
        }
    }
}
```

## 4. Run the application
---

``` powershell
    dotnet restore
    dotnet run
```

The output should read:
    
``` csharp
namespace CodeGenerationSample
{
    using System;

    public class Order : BaseEntity<Order>, IHaveIdentity
    {
        private bool canceled;
        public int Quantity
        {
            get;
            set;
        }

        public void MarkAsCanceled()
        {
            canceled = true;
        }
    }
}
```

Get a copy of the code here: <https://github.com/cmendible/dotnetcore.samples/tree/main/roslyn.create.class>

Hope it helps!