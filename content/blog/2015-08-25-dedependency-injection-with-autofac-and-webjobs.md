---
date: 2015-08-25T00:00:00Z
description: |
  Shows how you can use Autofac to resolved WebJob instances and inject dependencies into your web jobs
tags:
- azure webjobs
- autofac
- dependency injection
title: Dependency injection with Autofac and Azure WebJobs
url: /blog/dedependency-injection-with-autofac-and-webjobs/
---

## Introduction

At the end of last year [I wrote a blog post](http://www.jerriepelser.com/blog/using-autofac-and-common-service-locator-with-azure-webjobs) that demonstrated how you could use Autofac and the Common Service Locator in Azure WebJobs to resolve dependencies. At that time there was no way to inject dependencies, as Azure WebJobs functions had to be static methods and therefore there was no object instance into which dependencies could be injected. The Service Locator pattern worked fine for that scenario, but it was not necessarily an ideal situation.

Things have changed in the meantime and with the [release of version 1.0.1 of the WebJobs SDK](https://azure.microsoft.com/blog/2015/02/24/announcing-the-1-0-1-alpha-preview-of-microsoft-azure-webjobs-sdk/) you are no longer limited to static methods, as instance methods can also be triggered by the SDK.

In this blog post I will show how you can use Autofac to inject dependencies into your Web Jobs.

## Preparation

As per the previous blog post, I am once again building a web job that processes messages [sent to me by Dropbox using a web hook](http://www.jerriepelser.com/blog/creating-a-dropbox-webhook-in-aspnet). I have also created a very light wrapper around the Dropbox API using [Refit](https://github.com/paulcbetts/refit).

> If you want a quick introduction to Refit, [I have created a video](https://youtu.be/Myv7Hb90s5A) on my AspnetCasts YouTube channel which you can look at.

This is what my Dropbox API wrapper looks like:

``` csharp
public interface IDropboxApi
{
    [Post("/delta?path_prefix={path}&cursor={cursor}")]
    Task<DeltaResponse> GetDelta(string path, string cursor, [Header("Authorization")] string authorization);
}

public class DeltaResponse
{
    [JsonProperty("cursor")]
    public string Cursor { get; set; }

    [JsonProperty("entries")]
    public object[][] Entries { get; set; }

    [JsonProperty("has_more")]
    public bool HasMore { get; set; }

    [JsonProperty("reset")]
    public bool Reset { get; set; }
}
```

And currently the web job which processes the messages received from Dropbox looks as follows:

``` csharp
public class WebhooksWebJob
{
    public void ProcessDropboxWebhook([QueueTrigger("dropbox-webhook")] string notification, TextWriter log)
    {
        // do some stuff with the Dropbox API
        // I want to access IDropboxApi inside here...
    }
}
```

Note in the code above that I have created the WebJob function as an instance method (i.e. it is not `static`).

## Setting up dependency injection

For the dependency injection I will be using Autofac. So first install Autofac:

``` text
install-package autofac
```

We will need an implementation of `IJobActivator` which will resolve instances from an Autofac container. So create a class called AutofacJobActivator:

``` csharp
public class AutofacJobActivator : IJobActivator
{
    private readonly IContainer _container;

    public AutofacJobActivator(IContainer container)
    {
        _container = container;
    }

    public T CreateInstance<T>()
    {
        return _container.Resolve<T>();
    }
}
```

The implementation of this class is straight forward. It takes an instance of an Autofac `IContainer` through the constructor, and then resolve any dependencies through that container.

Next we need to set up the container inside the `Main()` method of our application. For references, this is what it currently looks like:

``` csharp
static void Main()
{
    var host = new JobHost();
    host.RunAndBlock();
}
```

First off create an Autofac `ContainerBuilder` and register `IDropboxApi` to return an instance by calling the Refit `RestService.For<>()` method. This will ensure that whenever a dependency for `IDropboxApi` is requested that it will return the implementation which was automatically created by Refit.

```csharp
var builder = new ContainerBuilder();
builder.Register(c =>
                 {
                     var httpClient = new HttpClient()
                     {
                         BaseAddress = new Uri("https://api.dropbox.com/1")
                     };
                     return RestService.For<IDropboxApi>(httpClient);
                 })
    .As<IDropboxApi>()
    .InstancePerDependency();
}
```

You will also need to ensure that you register the actual WebJob class with the container:

``` csharp
// Need to register webjob class in Autofac as well
builder.RegisterType<WebhooksWebJob>()
    .InstancePerDependency();
```
 
Finally create a new `JobHostConfiguration` and set the `JobActivator` to our new `AutofacJobActivator`, passing along the actual Autofac container by calling `builder.Build()`:

``` csharp
var jobHostConfiguration = new JobHostConfiguration
{
    JobActivator = new AutofacJobActivator(builder.Build())
};

var host = new JobHost(jobHostConfiguration);
host.RunAndBlock();
``` 

This is what the final implementation of my `Main()` method looks like:

``` csharp
static void Main()
{
    // Register IDropbox API in Autofac
    var builder = new ContainerBuilder();
    builder.Register(c =>
                        {
                            var httpClient = new HttpClient()
                            {
                                BaseAddress = new Uri("https://api.dropbox.com/1")
                            };
                            return RestService.For<IDropboxApi>(httpClient);
                        })
        .As<IDropboxApi>()
        .InstancePerDependency();

    // Need to register webjob class in Autofac as well
    builder.RegisterType<WebhooksWebJob>()
        .InstancePerDependency();


    var jobHostConfiguration = new JobHostConfiguration
    {
        JobActivator = new AutofacJobActivator(builder.Build())
    };

    var host = new JobHost(jobHostConfiguration);
    host.RunAndBlock();
}
```

And now I can change my `WebhooksWebJob` class to accept a parameter of type `IDropboxApi` through its constructor. Because an instance of `WebhooksWebJob` will now be created through the `AutofacJobActivator` class, and therefore resolved through the Autofac container, the dependencies will be injected correctly.
 
``` csharp
public class WebhooksWebJob
{
    private readonly IDropboxApi _dropboxApi;

    public WebhooksWebJob(IDropboxApi dropboxApi)
    {
        _dropboxApi = dropboxApi;
    }

    public void ProcessDropboxWebhook([QueueTrigger("dropbox-webhook")] string notification, TextWriter log)
    {
        // do some stuff with the Dropbox API...
        _dropboxApi.GetDelta(...);
    }
}
```
