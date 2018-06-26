---
date: 2016-11-24T00:00:00Z
description: |
  Demonstrates how you can use the .NET Core Configuration system in .NET Core unit test projects
tags:
- .net core
- unit testing
title: Using Configuration files in .NET Core Unit Test Projects
url: /blog/using-configuration-files-in-dotnet-core-unit-tests/
---

So another thing I came across while converting the Integration tests for the [Auth0.NET](https://github.com/auth0/auth0.net) SDK to .NET Core was that I had to make use of configuration files which specify the settings so the Integration test can talk with Auth0.

Here are some of the basics which got it working for me...

## Add the configuration file

First, add a `client-secrets.json` file to the Integration test project, e.g.

```json
{
  "AUTH0_CLIENT_ID": "...",
  "AUTH0_CLIENT_SECRET": "..."
}
```

Configure the `client-secrets.json` file to be copied to the output directory by updating the `buildOptions` in the `project.json` file:

```json
{
  "version": "1.0.0-*",

  "buildOptions": {
    "copyToOutput": {
      "include": [
        "client-secrets.json"
      ]
    }
  },

  "dependencies": {
    "..."
  },

  "testRunner": "nunit",

  "frameworks": {
    "net461": {}
  }}
```

## Include the .NET Core Configuration NuGet package

Include the JSON Configuration file NuGet package (`Microsoft.Extensions.Configuration.Json`) in your `project.json`

```json
{
  "version": "1.0.0-*",

  "buildOptions": {
    "copyToOutput": {
      "include": [
        "client-secrets.json"
      ]
    }
  },

  "dependencies": {
    "...",
    "Microsoft.Extensions.Configuration.Json": "1.0.0"
  },

  "testRunner": "nunit",

  "frameworks": {
    "net461": {}
  }
}
```

Be sure to run `dotnet restore` after you have added the package.

## Use the configuration in your unit tests

You can now use the configuration file in your unit tests by using the `ConfigurationBuilder` class:

```csharp
var config = new ConfigurationBuilder()
    .AddJsonFile("client-secrets.json")
    .Build();
```

And then access any configuration value: 

```csharp
var clientId = config["AUTH0_CLIENT_ID"]
```

You can read more about how configuration works in .NET Core projects in the [ASP.NET Core Configuration documentation](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration)