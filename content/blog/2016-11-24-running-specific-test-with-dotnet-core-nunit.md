---
date: 2016-11-24T00:00:00Z
description: |
  Demonstrates how to run a single unit test, or sets of unit tests, when using the NUnit 3 test runner for .NET Core.
tags:
- .net core
- nunit
- unit testing
title: Running a specific test with .NET Core and NUnit
url: /blog/running-specific-test-with-dotnet-core-nunit/
---

I converted the Unit tests for the [Auth0.NET](https://github.com/auth0/auth0.net) SDK to .NET Core. Currently the unit testing framework being used is NUnit, and NUnit 3 comes with a [test runner for .NET Core](https://github.com/nunit/dotnet-test-nunit).

You can make use of it by configuring your `project.json` as follows:

```json
{
    "version": "1.0.0-*",

    "dependencies": {
        "NUnit": "3.5.0",
        "dotnet-test-nunit": "3.4.0-beta-3"
    },

    "testRunner": "nunit",

    "frameworks": {
        "netcoreapp1.0": {
            "imports": "portable-net45+win8",
            "dependencies": {
                "Microsoft.NETCore.App": {
                    "version": "1.0.0-*",
                    "type": "platform"
                }
            }
        }
    }
}
```

> The configuration above is current as of the writing of this blog post. Please refer to the [NUnit 3 Test Runner for .NET Core GitHub Page](https://github.com/nunit/dotnet-test-nunit) to obtain the must up to date informaton on how to configure it. 

With this in place you can easily run your unit tests from the command line by simply running the command

```text
dotnet test
```

This will however run all the tests in a particular assembly (except for the `Explicit` ones), but what if you want to run only a specific unit test?

Well for that you can refer to the documentation for the [Console Command Line](https://github.com/nunit/docs/wiki/Console-Command-Line). According to that documentation, one of the parameters you can pass to the Console Runner is `--test`, which allows you to specify a comma-separated list of names of test to run. 

You can also pass this `--test` parameter to the `dotnet` test runner, which it seems is then passing it on to the NUnit .NET Core Test runner. So for example, if I wanted to run the unit test `Auth0.ManagementApi.IntegrationTests.UsersTests.Test_users_crud_sequence`, I could execute the following command:

```text
dotnet test --test Auth0.ManagementApi.IntegrationTests.UsersTests.Test_users_crud_sequence
```

And that will then only run that particular unit test:

![](/assets/images/2016-11-24-running-specific-test-with-dotnet-core-nunit/nunit-console-output.png)