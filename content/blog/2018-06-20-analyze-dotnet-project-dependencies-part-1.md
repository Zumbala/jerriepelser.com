---
title: "Analyzing .NET Core project dependencies: Finding Package References"
description:
  Demonstrate how to use the .NET Core CLI to analyze dependencies for any .NET Core project.
tags:
- .net core
date: 2018-06-20
---

When developing [dotnet-outdated](https://github.com/jerriep/dotnet-outdated) I had to find a way to determine the packages referenced by a project. At first, I was using [BuildAlyzer](https://github.com/daveaglick/Buildalyzer), but after a suggestion from a user, I decided to look into using the .NET Core CLI itself to generate a dependency graph.

The dependency graph will indicate all the dependencies for a project that needs to be restored, and also contains useful additional information such as the target frameworks for the project and the NuGet package sources which should be used for restoring NuGet packages and which, in the case of **dotnet-outdated**, should be used for scanning for newer versions of said packages.

Another useful feature of the dependency graph is that, when used against a solution file, it will contain the output for all the projects in the solution.

To generate a dependency graph, you can use the [dotnet msbuild](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-msbuild) command, passing the `/t:GenerateRestoreGraphFile` and `/p:RestoreGraphOutputPath` parameters. 

For example, running the following command a the directory containing a .NET Core/.NET Standard project or solution will output a dependency graph file called `graph.dg`:

```text
dotnet msbuild /t:GenerateRestoreGraphFile /p:RestoreGraphOutputPath=graph.dg
```

The sample command above will try and locate a solution or project in the directory from which it is run, but you can also pass it the path to a solution or project:

```text
dotnet msbuild MySolution.sln /t:GenerateRestoreGraphFile /p:RestoreGraphOutputPath=graph.dg
```

## Structure of the dependency graph

Below is the sample dependency graph output generated for the project accompanying this blog post:

```json
{
  "format": 1,
  "restore": {
    "C:\\Development\\jerriepelser-blog\\AnalyzeDotNetProject\\AnalyzeDotNetProject.csproj": {}
  },
  "projects": {
    "C:\\Development\\jerriepelser-blog\\AnalyzeDotNetProject\\AnalyzeDotNetProject.csproj": {
      "version": "1.0.0",
      "restore": {
        "projectUniqueName": "C:\\Development\\jerriepelser-blog\\AnalyzeDotNetProject\\AnalyzeDotNetProject.csproj",
        "projectName": "AnalyzeDotNetProject",
        "projectPath": "C:\\Development\\jerriepelser-blog\\AnalyzeDotNetProject\\AnalyzeDotNetProject.csproj",
        "packagesPath": "C:\\Users\\jerri\\.nuget\\packages\\",
        "outputPath": "C:\\Development\\jerriepelser-blog\\AnalyzeDotNetProject\\obj\\",
        "projectStyle": "PackageReference",
        "fallbackFolders": [
          "C:\\Program Files\\dotnet\\sdk\\NuGetFallbackFolder"
        ],
        "configFilePaths": [
          "C:\\Users\\jerri\\AppData\\Roaming\\NuGet\\NuGet.Config",
          "C:\\Program Files (x86)\\NuGet\\Config\\Microsoft.VisualStudio.Offline.config"
        ],
        "originalTargetFrameworks": [
          "netcoreapp2.1"
        ],
        "sources": {
          "C:\\Program Files (x86)\\Microsoft SDKs\\NuGetPackages\\": {},
          "https://api.nuget.org/v3/index.json": {}
        },
        "frameworks": {
          "netcoreapp2.1": {
            "projectReferences": {}
          }
        },
        "warningProperties": {
          "warnAsError": [
            "NU1605"
          ]
        }
      },
      "frameworks": {
        "netcoreapp2.1": {
          "dependencies": {
            "Microsoft.NETCore.App": {
              "target": "Package",
              "version": "[2.1.0, )",
              "autoReferenced": true
            },
            "NuGet.ProjectModel": {
              "target": "Package",
              "version": "[4.7.0, )"
            },
            "newtonsoft.json": {
              "target": "Package",
              "version": "[11.0.2, )"
            }
          },
          "imports": [
            "net461"
          ],
          "assetTargetFallback": true,
          "warn": true
        }
      }
    }
  }
}
```

Under the `projects` node, you will notice an entry for the actual project. In the case of multiple projects (such as when generating it against a solution file) the `projects` node will contain separate child nodes for each project.

The `sources` node indicates the list of NuGet sources which should be used when restoring NuGet packages. These can also be used to scan for updates of said NuGet packages.

The `frameworks` node will contain an entry for each target framework of the project, and below that, under the `dependencies` node, you will find entries for each package reference along with the version of the package.

## Processing the dependency graph

Now that we have a dependency graph, all we need to do is to process that JSON file to understand the actual structure of a solution with its projects, as well as the target framework for each project along with the dependencies for that target framework.

You could process it yourself using a library such as [JSON.NET](https://www.newtonsoft.com/json), but it turns out that the [NuGet client libraries](https://github.com/NuGet/NuGet.Client) already contain .NET classes to help you process the dependency graph.

Add the [NuGet.ProjectModel](https://www.nuget.org/packages/NuGet.ProjectModel) package to your project. Also add the **Newtonsoft.Json** package:

```text
dotnet add package NuGet.ProjectModel
dotnet add package Newtonsoft.Json
```

Now you can to load the contents of the dependency graph from the generated file, deserialise the JSON into a `JObject` and then pass that to the constructor of the `DependencyGraphSpec` class.

```csharp
string dependencyGraphText = File.ReadAllText(dgOutput);
var dependencyGraph = new DependencyGraphSpec(JsonConvert.DeserializeObject<JObject>(dependencyGraphText));
```

The `DependencyGraphSpec` instance allows you to iterate through the projects, for each project its target frameworks and for each target framework its references:

```csharp
foreach(var project in dependencyGraph.Projects.Where(p => p.RestoreMetadata.ProjectStyle == ProjectStyle.PackageReference))
{
    Console.WriteLine(project.Name);
    
    foreach(var targetFramework in project.TargetFrameworks)
    {
        Console.WriteLine($"  [{targetFramework.FrameworkName}]");

        foreach(var dependency in targetFramework.Dependencies)
        {
            Console.WriteLine($"  {dependency.Name}, v{dependency.LibraryRange.VersionRange.ToShortString()}");
        }
    }
}
```

When processing the [source code for dotnet-outdated](https://github.com/jerriep/dotnet-outdated), it will produce the following output:

![Output when running application](/images/blog/2018-06-20-analyze-dotnet-project-dependencies-part-1/output.png)

In the [next blog post](/blog/analyze-dotnet-project-dependencies-part-2/), I'll dive a little bit deeper and look at how one can determine the transitive dependencies for a project.

I put together a sample application demonstrating these techniques, which you can find at [https://github.com/jerriepelser-blog/AnalyzeDotNetProject](https://github.com/jerriepelser-blog/AnalyzeDotNetProject).