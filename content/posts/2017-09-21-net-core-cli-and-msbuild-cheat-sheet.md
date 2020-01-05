---
author: Carlos Mendible
categories:
- dotnetcore
crosspost_to_medium: true
date: "2017-09-21T17:23:14Z"
description: A small .NET Core CLI and MSBUILD Cheat Sheet
image: /wp-content/uploads/2017/07/dotnetcore.png
# tags: CheatSheet CLI msbuild
title: .NET Core CLI and MSBUILD Cheat Sheet
---
This is a small **.NET Core CLI and MSBUILD Cheat Sheet** with the list of commands and settings I use almost on daily basis when working with .NET Core, the command line and Visual Studio Code.

## Checks
### Check installed CLI version:
    
``` powershell
dotnet --version
``` 

### Show available templates:

``` powershell
dotnet new
``` 

## Solutions
### Create a solution with the name of current the folder: 
    
``` powershell
dotnet new sln
```
    
### Create a solution with a specific name:

``` powershell
dotnet new sln --name [solution name]
```
    
### Add a project to a solution:

``` powershell
dotnet add sln [relative path to csproj file]
```

## Packages
### Add package reference to the project in the current folder:
    
``` powershell
dotnet add package [package name]
```
    
### Remove a package reference from the project in the current folder:
``` powershell
dotnet remove package [package name]
```
    
### Add a specific package version reference to the project in the current folder:

``` powershell
dotnet add package [package name]-v [version]
```
    
### Restore packages:

``` powershell
dotnet restore
```
    
### Create a nuget package for the project in current folder:

``` powershell
dotnet pack
```

## Project Templates
### Install a new project template:
    
``` powershell
dotnet new --install [dotnet template name]
```
    
### Remove a project template:

``` powershell
dotnet new --uninstall [dotnet template name]
```

###  Run test defined in current folder project
    
``` powershell
dotnet test
``` 

## Builds
### Build current's folder solution or project
    
``` powershell
dotnet build
```

### Build current's folder solution or project with release configuration

``` powershell
dotnet build -c Release
```

### Publish artifacts for current's folder solution or project.

``` powershell
dotnet publish 
```

## MSBUILD
### To add a reference to a local assembly without nuget package, edit your csproj and add a Reference as shown in the following sample:
    
``` xml
<code>
  <ItemGroup>
    <Reference Include="[Relative path to the Assembly dll file]" />
  </ItemGroup>
</code>
```
    
### Force a file to be copied to the output folder, edit your csproj and add a Content section as shown in the following sample:
    
``` xml
<code>
 <ItemGroup>
  <Content Include="[name of the file]">
   <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </Content>
 </ItemGroup>
</code>
```

Hope it helps!