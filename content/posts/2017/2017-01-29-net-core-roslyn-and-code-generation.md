---
author: Carlos Mendible
categories:
- dotnet
crosspost_to_medium: true
date: "2017-01-29T20:31:41Z"
description: .NET Core, Roslyn and Code Generation
images: ["/wp-content/uploads/2017/01/roslyn.png"]
tags: ["Roslyn"]
title: .NET Core, Roslyn and Code Generation
url: /2017/01/29/net-core-roslyn-and-code-generation/
---
For ages I've been using T4 templates as main tool for code generation and scaffolding, but now that I'm an absolute fan of Visual Studio Code and .Net Core I need to explore other options such as Yeoman, Scripty and Roslyn. This post is just the result of my first and simplest experiment with **.Net Core, Roslyn and Code Generation**.

In the following sample we will take a class and replace the namespace with another. My intention is just to show you the tip of the iceberg of what you can accomplish with these tools!

## 1. Create the application
---
  
Open a command prompt and run 
    
``` powershell
    md roslyn.codegeneration
    cd roslyn.codegeneration
    dotnet new
    dotnet restore
    code .
```

## 2. Replace the contents of project.json
---
Replace the contents on **project.json** file in order to include the references to both: **Microsoft.CodeAnalysis.CSharp** and **Microsoft.CodeAnalysis.Compilers**.
    
``` json
{
  "version": "1.0.0-*",
  "buildOptions": {
    "debugType": "portable",
    "emitEntryPoint": true
  },
  "dependencies": {
    "Microsoft.CodeAnalysis.CSharp": "2.0.0-rc2",
    "Microsoft.CodeAnalysis.Compilers": "2.0.0-rc2"
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
The **ChangeNamespaceAsync** method is where the magic occurs. It takes the code in a string, parses it, replaces the namespace and finally writes the class to the console and a file.
    
``` csharp
using System;
using System.IO;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;

namespace Roslyn.CodeGeneration
{
    public class Program
    {
        public static void Main(string[] args)
        {
            // We will change the namespace of this sample code.
            var code =
            @"  using System; 

                namespace OldNamespace 
                { 
                    public class Person
                    {
                        public string Name { get; set; }
                        public int Age {get; set; }
                    }
                }";

            // Use Task to call ChangeNamespaceAsync
            Task.Run(async () =>
            {
                await ChangeNamespaceAsync(code, "NamespaceChangedUsingRoslyn");
            })
            .GetAwaiter()
            .GetResult();

            // Wait to exit.
            Console.Read();
        }

        /// <summary>
        /// Changes the namespace for the given code.
        /// </summary>
        static async Task ChangeNamespaceAsync(string code, string @namespace)
        {
            // Parse the code into a SyntaxTree.
            var tree = CSharpSyntaxTree.ParseText(code);

            // Get the root CompilationUnitSyntax.
            var root = await tree.GetRootAsync().ConfigureAwait(false) as CompilationUnitSyntax;

            // Get the namespace declaration.
            var oldNamespace = root.Members.Single(m => m is NamespaceDeclarationSyntax) as NamespaceDeclarationSyntax;

            // Get all class declarations inside the namespace.
            var classDeclarations = oldNamespace.Members.Where(m => m is ClassDeclarationSyntax);

            // Create a new namespace declaration.
            var newNamespace = SyntaxFactory.NamespaceDeclaration(SyntaxFactory.ParseName(@namespace)).NormalizeWhitespace();

            // Add the class declarations to the new namespace.
            newNamespace = newNamespace.AddMembers(classDeclarations.Cast<MemberDeclarationSyntax>().ToArray());

            // Replace the oldNamespace with the newNamespace and normailize.
            root = root.ReplaceNode(oldNamespace, newNamespace).NormalizeWhitespace();

            string newCode = root.ToFullString();

            // Write the new file.
            File.WriteAllText("Person.cs", root.ToFullString());

            // Output new code to the console.
            Console.WriteLine(newCode);
            Console.WriteLine("Namespace replaced...");
        }
    }
}
```

## 4. Run the application
---
Open a command prompt and run 
    
``` powershell
    dotnet run
```
    
The output should read:
    
``` csharp
using System;

namespace NamespaceChangedUsingRoslyn
{
    public class Person
    {
        public string Name
        {
            get;
            set;
        }

        public int Age
        {
            get;
            set;
        }
    }
}
Namespace replaced...
```

Get a copy of the code here: <https://github.com/cmendible/dotnetcore.samples/tree/master/roslyn.codegeneration>

Hope it helps!