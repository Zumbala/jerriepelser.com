---
title: "Using version ranges for package dependencies in .NET project files"
description:
  When adding package references in your projects files, you are not limited to specifying a specific version. You can specify a version range instead.
tags:
- dotnet
- dotnet core
- nuget
- visual studio
- rider
---

The one nice side-effect of working on [dotnet-outdated](https://github.com/jerriep/dotnet-outdated) is that I am learning a lot of interesting things about working with NuGet packages in your projects.

In trying to make sense of how the .NET tooling resolves which NuGet package to use, I came across an interesting bit in the NuGet documentation about [Package Versioning](https://docs.microsoft.com/en-us/nuget/reference/package-versioning). This document talks about how you can specify version ranges for packages, and [in the examples](https://docs.microsoft.com/en-us/nuget/reference/package-versioning#examples) it shows how you can use a version range in a `PackageReference` in your `.csproj` file.

**This was news to me.**

I guess it is because I am so used to using the NuGet tooling in Visual Studio and Rider, or the `dotnet add package` command, to install NuGet packages. Using this tooling, you will typically install a specific version of a package.

## Using a wildcard

Let me demonstrate how using a version range works in practice by making use of version number wildcards. For this blog post, I'll be playing around with the reference to the [McMaster.Extensions.CommandLineUtils NuGet package](https://www.nuget.org/packages/McMaster.Extensions.CommandLineUtils). If we look at the NuGet page for this package, we can see the following available versions of the package:

![Available versions for McMaster.Extensions.CommandLineUtils](/images/blog/2018-05-25-using-version-range-package-dependencies-project-files/commandlineutils-versions.png)

Let's edit the `.csproj` file for the [dotnet-outdated project](https://github.com/jerriep/dotnet-outdated) and specify that we are willing to take the latest package with the **major** version of **2**:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <!-- rest of project file contents omitted -->
  <ItemGroup>
    <PackageReference Include="McMaster.Extensions.CommandLineUtils" Version="2.*" />
    <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="2.1.0-rc1-final" />
    <PackageReference Include="Newtonsoft.Json" Version="11.0.2" />
    <PackageReference Include="NuGet.ProjectModel" Version="4.8.0-preview1.5156" />
    <PackageReference Include="NuGet.Protocol" Version="4.8.0-preview1.5156" />
    <PackageReference Include="NuGet.Versioning" Version="4.8.0-preview1.5156" />
    <PackageReference Include="System.IO.Abstractions" Version="2.1.0.178" />
  </ItemGroup>
</Project>
```

When I load the project inside Visual Studio, you can see that it correctly resolves to the latest version (of the major version 2):

![Dependencies in the Visual Studio Solution Explorer](/images/blog/2018-05-25-using-version-range-package-dependencies-project-files/visual-studio-dependencies-1.png)

Same thing for Rider:

![Dependencies in the Rider Solution Explorer](/images/blog/2018-05-25-using-version-range-package-dependencies-project-files/rider-dependencies-1.png)

Let's say I want to use the latest version in the **2.1** range; I can update my `PackageReference` as follows:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <!-- rest of project file contents omitted -->
  <ItemGroup>
    <PackageReference Include="McMaster.Extensions.CommandLineUtils" Version="2.1.*" />
    <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="2.1.0-rc1-final" />
    <PackageReference Include="Newtonsoft.Json" Version="11.0.2" />
    <PackageReference Include="NuGet.ProjectModel" Version="4.8.0-preview1.5156" />
    <PackageReference Include="NuGet.Protocol" Version="4.8.0-preview1.5156" />
    <PackageReference Include="NuGet.Versioning" Version="4.8.0-preview1.5156" />
    <PackageReference Include="System.IO.Abstractions" Version="2.1.0.178" />
  </ItemGroup>
</Project>
```

Once again, both IDEs will resolve the correct version. Displayed below is a screenshot from Visual Studio for this configuration:

![Dependencies in the Visual Studio Solution Explorer](/images/blog/2018-05-25-using-version-range-package-dependencies-project-files/visual-studio-dependencies-2.png)

## Problems with the tooling

The one thing where the IDE tooling falls a bit short, however, is in the Package Manager (for Visual Studio) and NuGet Tool Window (for Rider).

Let's go back to the first instance where I specified a range of __2.*__. Even though the referenced version shows correctly as **2.2.4** in the Solution Explorer (which is the latest version), the NuGet Package Manager shows the version as **2.0.0** and suggests that an update to **2.2.4** is available.

![NuGet Package Manager](/images/blog/2018-05-25-using-version-range-package-dependencies-project-files/nuget-package-manager.png)

The same thing goes for Rider:

![NuGet Tool Window](/images/blog/2018-05-25-using-version-range-package-dependencies-project-files/nuget-tool-window.png)

## Version ranges in dotnet-outdated

BTW, the upcoming version (0.3.0) of **dotnet-outdated** will do the right thing. It uses the same NuGet SDK and classes as the NuGet tooling, so it will also detect the correct referenced version when using version ranges. In the sample screenshot below, I ran it agains the project file which contains a `PackageReference` of __2.*__, and it displays the referenced version correctly as **2.2.4**:

![Version ranges in dotnet-outdated](/images/blog/2018-05-25-using-version-range-package-dependencies-project-files/dotnet-outdated.png)

## Conclusion

Using version ranges in your project is a useful way to ensure that you are always on the latest version of a package. This may not be desirable, but by using wildcards and upper version range, you can ensure that it does not accidentally update you to a next major version for example.