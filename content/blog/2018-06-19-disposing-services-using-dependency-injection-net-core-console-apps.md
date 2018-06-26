---
title: "Disposing of services when using Dependency Injection with .NET Core console apps"
description:
  Look at how you can ensure that services are disposed of when using dependency injection with .NET Core Console applications.
tags:
- .net core
- dependency injection
---

## Background

The [Microsoft documentation on Dependency Injection](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection) is primarily focused on using it with an ASP.NET Core application, but as demonstrated previously by Andrew Lock, [you can use Dependency Injection with a regular .NET Core console application](https://andrewlock.net/using-dependency-injection-in-a-net-core-console-application/).

Borrowing from Andrew's sample, let's say we have the following service:

```csharp
public interface IFooService
{
    void DoSomething();
}

public class FooService : IFooService
{
    public void DoSomething()
    {
        // Do some work here
    }
}
```

I can register it with DI and then obtain a reference to the implementation from the service provider:

```csharp
class Program
{
    static void Main(string[] args)
    {
        // Configure DI
        var serviceProvider = new ServiceCollection()
            .AddSingleton<IFooService, FooService>()
            .BuildServiceProvider();
    
        // Do the work..
        var foo = serviceProvider.GetService<IFooService>();
        foo.DoSomething();
    }
}
```

This is a very simplistic example, but you should get the idea.

In my particular case, I was using dependency injection with Nate McMaster's CommandLineUtils which allows you to [inject services into the constructors](https://natemcmaster.github.io/CommandLineUtils/docs/concepts/dependency-injection.html) of your commands.

Once again, a very simplistic example of that would look something like this:

```csharp
[Command(Name = "di", Description = "Dependency Injection sample project")]
[HelpOption]
class Program
{
    public static int Main(string[] args)
    {
        var serviceProvider = new ServiceCollection()
            .AddSingleton<IFooService, FooService>()
            .BuildServiceProvider();

        var app = new CommandLineApplication<Program>();
        app.Conventions
            .UseDefaultConventions()
            .UseConstructorInjection(serviceProvider);

        return app.Execute(args);
    }

    private readonly IFooService _fooService;

    public Program(IFooService fooService)
    {
        _fooService = fooService;
    }

    private void OnExecute()
    {
        _fooService.DoSomething();
    }
}
```

The sample above will create a service provider with the required services and then pass that along to **CommandLineUtils** using the constructor injection convention. The result is that **CommandLineUtils** will inject the correct implementation of `IFooService` into the constructor of my `Program` class.

## Disposing services

So far, so good. The issue I ran into, however, when developing [dotnet-outdated](https://github.com/jerriep/dotnet-outdated), was that some of my services needed to clean up after themselves. They created temporary files which they had to delete, and I achieved this by implementing the `IDisposable` interface.

Expanding on the previous sample, here is an update of my `FooService` class, implementing `IDisposable`:

```csharp
public interface IFooService
{
    void DoSomething();
}

public class FooService : IFooService, IDisposable
{
    public void DoSomething()
    {
        // Do some work here
    }

    public void Dispose()
    {
        // Clean up resources...
    }
}
```

The next problem I ran into was that the object never gets disposed. At first, I thought I need to keep a reference to it and dispose of it manually, but it turns out that the `ServiceProvider` class itself also implements `IDisposble`:

```csharp
public sealed class ServiceProvider : IServiceProvider, IDisposable, IServiceProviderEngineCallback
{
    public void Dispose();

    public object GetService(Type serviceType);
}
```

When disposing of the service provider, it will, in turn, dispose of all objects it is managing that requires disposing of. So with that in mind, all I needed to do was to dispose of the service provider reference:

```csharp
public static int Main(string[] args)
{
    using (var serviceProvider = new ServiceCollection()
        .AddSingleton<IFooService, FooService>()
        .BuildServiceProvider())
    {
        var app = new CommandLineApplication<Program>();
        app.Conventions
            .UseDefaultConventions()
            .UseConstructorInjection(serviceProvider);

        return app.Execute(args);
    }
}
```

## Conclusion

This blog post demonstrated how you could use the [dispose pattern](https://docs.microsoft.com/en-us/dotnet/standard/design-guidelines/dispose-pattern) when using dependency injection with a regular .NET Core console application.