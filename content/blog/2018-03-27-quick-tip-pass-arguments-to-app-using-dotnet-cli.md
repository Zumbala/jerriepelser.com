---
title: "Quick Tip: Pass arguments to your app when using the .NET CLI"
description: |
  A quick tip to demonstrate how you can pass arguments to your application using the .NET CLI
date: 2018-03-27
tags:
- cli
- dotnet
- dotnet cli
url: /blog/quick-tip-pass-arguments-to-app-using-dotnet-cli
---

I am playing around with developing a .NET Core Global tool (which is [coming in .NET Core 2.1](https://blogs.msdn.microsoft.com/dotnet/2018/02/02/net-core-2-1-roadmap/)), and I am allowing users to pass command line arguments to my tool which I parse using Nate McMaster's [CommandLineUtils](https://github.com/natemcmaster/CommandLineUtils) library.

At this stage of development I am not installing my app as a Global Tool yet, but instead just running it using the standard `dotnet run` command.

One of the options a user can pass to my app is a `-h` option which will print out the list of commands. So, I ran `dotnet run -h`, and this is the output I saw:

![](/images/blog/2018-03-27-quick-tip-pass-arguments-to-app-using-dotnet-cli/dotnet-cli-help.png)

As you will notice, that is not the help text from my application, but instead the help text from the .NET Core CLI. To pass arguments to your application, you need to pass a `--` argument, and then the arguments to your application. As per the [.NET Core CLI documentation](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-run?tabs=netcore2x#options), _`--` delimits arguments to `dotnet run` from arguments for the application being run. All arguments after this one are passed to the application run_.

So in other words, if I want to pass the `-h` argument to my application, I need to do it as follows:

```
dotnet run -- -h
```

And with that, I get the correct output:

![](/images/blog/2018-03-27-quick-tip-pass-arguments-to-app-using-dotnet-cli/dotnet-app-help.png)