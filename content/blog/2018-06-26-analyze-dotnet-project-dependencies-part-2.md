---
title: "Analyzing .NET Core project dependencies: Finding transitive dependencies"
description:
  Demonstrate how to use the .NET Core CLI to find transitive dependencies for any .NET Core project.
tags:
- .net core
date: 2018-06-26
---

## Introduction

In the [previous blog post](/blog/analyze-dotnet-project-dependencies-part-1/), I looked at how we can use the .NET Core CLI to generate a dependency graph which allows you to determine the package references for a .NET Core or .NET Standard project. This is the same technique I used when developing [dotnet-outdated](https://github.com/jerriep/dotnet-outdated).

Once **dotnet-outdated** started gaining a bit of traction, one of the issues opened on the GitHub repository was a request to [support detecting outdated transitive dependencies](https://github.com/jerriep/dotnet-outdated/issues/13).

## Overview of transitive dependencies

_So what are transitive dependencies?_

Well, let's take the [source code for dotnet-outdated](https://github.com/jerriep/dotnet-outdated) as an example again. As we could see from the sample application in the previous blog post, one of the packages I reference in that project is the [McMaster.Extensions.CommandLineUtils](https://www.nuget.org/packages/McMaster.Extensions.CommandLineUtils) package:

![Output from the application in the previous blog post](/images/blog/2018-06-26-analyze-dotnet-project-dependencies-part-2/output-1.png)

If we look at the [McMaster.Extensions.CommandLineUtils NuGet page](https://www.nuget.org/packages/McMaster.Extensions.CommandLineUtils) we will notice that it, in turn, depends on a number of packages, one of which is `System.ComponentModel.Annotations`:

![Dependencies of McMaster.Extensions.CommandLineUtils](/images/blog/2018-06-26-analyze-dotnet-project-dependencies-part-2/nuget-1.png)

In this example, `System.ComponentModel.Annotations` is what is referred to as a _transitive dependency_. This means that it will be referenced by our application _transitively_ because another package our application references depends on it.

In particular, it depends on **v4.4.1** or higher of that package. The way NuGet works is that it will, by default, reference the minimum required version satisfying that criteria - in other words,**v4.4.1**.

If we look at the [NuGet page for System.ComponentModel.Annotations](https://www.nuget.org/packages/System.ComponentModel.Annotations/) we will see that a newer version of that package is available:

![Available versions of System.ComponentModel.Annotations](/images/blog/2018-06-26-analyze-dotnet-project-dependencies-part-2/nuget-2.png)

So this is what that GitHub issue requested. They wanted **dotnet-outdated** to report that even though out application references **v4.4.1** of `System.ComponentModel.Annotations`, a newer version of that package existed. It would then be up to the developer to reference that version directly if they rather want to use the newer version, instead of the older version which was _transitively_ referenced.

In the rest of this blog post I will demonstrate how to determine the _transitive dependencies_ for an application. Determining whether or not that dependency is outdated is not in the scope of this blog post.

## Determining transitive dependencies

OK, so now that we know what a transitive dependency is, _how do we find them_?

My first thought was that I would need to determine these using the NuGet APIs. In fact, soon after that GitHub issue was opened, someone [submitted a pull request](https://github.com/jerriep/dotnet-outdated/pull/24) doing it this way.

I accepted that PR but started looking for a better way to do it. I was determined to find out whether I could not again piggy-back on work the .NET Core CLI is doing - same as I did for the dependency graph.

_It turned out there was indeed a way to do this with the .NET Core CLI._

When the .NET Core CLI restores packages, it creates a `project.assets.json` file which lists the dependencies of the application. This file is generated and placed in the project's build assets output path. This is typically the `/obj` directory of the project, but it can differ. Thankfully the dependency graph we generated last time contains this information in the `RestoreMetadata.OutputPath` property for a project.

Here is a simplified version of the `project.assets.json` file

```json
{
  "version": 3,
  "targets": {
    ".NETCoreApp,Version=v2.1": {
      "McMaster.Extensions.CommandLineUtils/2.2.4": {
        "type": "package",
        "dependencies": {
          "System.ComponentModel.Annotations": "4.4.1"
        },
        "compile": {
          "lib/netstandard2.0/McMaster.Extensions.CommandLineUtils.dll": {}
        },
        "runtime": {
          "lib/netstandard2.0/McMaster.Extensions.CommandLineUtils.dll": {}
        }
      },
      ...
      "System.ComponentModel.Annotations/4.4.1": {
        "type": "package",
        "compile": {
          "ref/netcoreapp2.0/_._": {}
        },
        "runtime": {
          "lib/netcoreapp2.0/_._": {}
        }
      }
      ...
  },
  "libraries": {
    ...
  },
  "projectFileDependencyGroups": {
    ...
  },
  "packageFolders": {
    ...
  },
  "project": {
    ...
  }
}
```

For our purpose, the important part of `project.assets.json` is the `targets` node. This contains an entry for each target framework of our application which in turn contain an entry for every package referenced by the application - whether it is a direct dependency or a transitive dependency. Each one of those dependencies can, in turn, have a `dependencies` node listing its dependencies.

So to get the proper dependency tree of an application, we will need to combine the contents of `project.assets.json` with the contents of the dependency graph from [last time](/blog/analyze-dotnet-project-dependencies-part-1/).

First, we will use the dependency graph to determine the _explicit dependencies_ of our application. The for each of those dependencies we'll head over to the `project.assets.json` and find the dependency under the relevant target framework. We will then look at the `dependencies` node for the dependency to find its _transitive dependencies_. Each transitive dependency can of course, in turn, have it own dependencies, so we will need to handle that recursively.

## The application logic

The first thing we'll need is a method which runs the `dotnet restore` command and then loads the `project.assets.json` file. For this, I created a `LockFileService` class. Same as last time with the dependency graph, the [NuGet.ProjectModel](https://www.nuget.org/packages/NuGet.ProjectModel) package contains a `LockFile` class which is a nice .NET wrapper around the `project.assets.json` file. We will also use the `LockFileUtilities` to load the contents of the `project.assets.json` file.

```csharp
public class LockFileService
{
    public LockFile GetLockFile(string projectPath, string outputPath)
    {
        // Run the restore command
        var dotNetRunner = new DotNetRunner();
        string[] arguments = new[] {"restore", $"\"{projectPath}\""};
        var runStatus = dotNetRunner.Run(Path.GetDirectoryName(projectPath), arguments);

        // Load the lock file
        string lockFilePath = Path.Combine(outputPath, "project.assets.json");
        return LockFileUtilities.GetLockFile(lockFilePath, NuGet.Common.NullLogger.Instance);
    }
}
```

Now we can update the logic from last time to locate each dependency from the `LockFile` instance and then call a `ReportDependency` method to print the name of the dependency to the console.

```csharp
foreach(var project in dependencyGraph.Projects.Where(p => p.RestoreMetadata.ProjectStyle == ProjectStyle.PackageReference))
{
    // Generate lock file
    var lockFileService = new LockFileService();
    var lockFile = lockFileService.GetLockFile(project.FilePath, project.RestoreMetadata.OutputPath);

    Console.WriteLine(project.Name);

    foreach(var targetFramework in project.TargetFrameworks)
    {
        Console.WriteLine($"  [{targetFramework.FrameworkName}]");

        var lockFileTargetFramework = lockFile.Targets.FirstOrDefault(t => t.TargetFramework.Equals(targetFramework.FrameworkName));
        if (lockFileTargetFramework != null)
        {
            foreach(var dependency in targetFramework.Dependencies)
            {
                var projectLibrary = lockFileTargetFramework.Libraries.FirstOrDefault(library => library.Name == dependency.Name);

                ReportDependency(projectLibrary, lockFileTargetFramework, 1);
            }
        }

    }
}
```

The `ReportDependency` method is called recursively for each of the child dependencies:

```csharp
private static void ReportDependency(LockFileTargetLibrary projectLibrary, LockFileTarget lockFileTargetFramework, int indentLevel)
{
    Console.Write(new String(' ', indentLevel * 2));
    Console.WriteLine($"{projectLibrary.Name}, v{projectLibrary.Version}");

    foreach (var childDependency in projectLibrary.Dependencies)
    {
        var childLibrary = lockFileTargetFramework.Libraries.FirstOrDefault(library => library.Name == childDependency.Id);

        ReportDependency(childLibrary, lockFileTargetFramework, indentLevel + 1);
    }
}
```

And this is the result:

![Output from the application](/images/blog/2018-06-26-analyze-dotnet-project-dependencies-part-2/output-2.png)

The actual output can become quite lengthy, but in the screenshot above you can see the hierarchical nature of the output as we report on each dependency, and for each dependency we report on its child dependencies and so on.

## Conclusion

This blog post demonstrated how you can use the standard assets generated by the .NET Core CLI to understand the structure of an application - in particular the dependencies of an application along with the transitive dependencies.

The sample application demonstrating these techniques can be found at [https://github.com/jerriepelser-blog/AnalyzeDotNetProject](https://github.com/jerriepelser-blog/AnalyzeDotNetProject).