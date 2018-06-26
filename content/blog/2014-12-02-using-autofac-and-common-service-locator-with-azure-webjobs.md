---
date: 2014-12-02T00:00:00Z
description: |
  Demonstrates how to use Autofac and the Common Service Locator to resolve dependencies in Azure WebJobs
tags:
- azure webjobs
- autofac
- service locator
title: Using Autofac and Common Service Locator with Azure WebJobs
url: /blog/using-autofac-and-common-service-locator-with-azure-webjobs/
---

I am busy developing an application which makes heavy use of Azure WebJobs. One of the things which I noticed right away was that there does not seem to be any built-in support in the Azure WebJobs SDK for dependency injection. I set about trying to find out how I can still use dependency injection, but it seems to be impossible with the current design and extensibility points in the SDK. One of the main problems is that the methods which gets executed by the SDK are `public static` methods, so there is no object instantiation and therefore no possibility for constructor injection.

The solution I came up with is to instead use the Service Locator pattern. I know it is described as an anti-pattern and we are told to avoid it, but hell if it solves my problem of loosely coupling dependencies and also allows me to create mocks for unit testing I won't complain or get hung up on it. 

Here is the solution I came up with and works for me. Your mileage may vary, so use it if you like it or leave it if you don't.

## Some background

For my application I have a WebJob which monitors a queue for webhook notifications from Dropbox ([which I discussed in my previous blog post](/blog/creating-a-dropbox-webhook-in-aspnet)). So a user updates a file in their Dropbox account, and Dropbox notifies my application via a Webhook. The Webhook on my side pushes the notification on to a queue which then gets processed by the WebJob. As part of the processing I want to be able to get the [Delta information using the Dropbox API](https://www.dropbox.com/developers/core/docs#delta).

So in my WebJob I need to integrate with the Dropbox API and for that I use the excellent [Refit](https://github.com/paulcbetts/refit) package from Paul Betts. So I have created a `IDropboxApi` interface with the methods I am interested in using:

```csharp
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

public interface IDropboxApi
{
    [Post("/delta?path_prefix={path}&cursor={cursor}")]
    Task<DeltaResponse> GetDelta(string path, string cursor, [Header("Authorization")] string authorization);
}
```

For the WebJob I have bound the input parameter to the queue using the `QueueTrigger` attribute, so whenever a new message is placed on that queue the WebJob gets triggered and the message which was placed on the queue gets passed in as the `notification` parameter:

```csharp
public class WebhooksWebJob
{
    public static void ProcessDropboxWebhook([QueueTrigger("dropbox-webhook")] string notification, TextWriter log)
    {
    }
}
```

So let us assume that at this point I want to be able to call the `IDropboxApi` interface to get the delta for each user who Dropbox notified me that their files have changed, so that I can get a list of the exact files which changed. For the sake of being able to unit test things nicely I do not want to create an instance of the `IDropboxApi` class in my WebJob, but like I said above I am also not able to inject it using dependency injection. 

## Enter the Service Locator

For the next parts you will need to install the Autofac and CommonServiceLocator Nuget Packages, so from the Nuget Package Console, install the following packages:

```text
Install-Package CommonServiceLocator

Install-Package Autofac

Install-Package Autofac.Extras.CommonServiceLocator
```

Next thing is that I change my WebJob so I can specify an `IServiceLocator` instance as a property:

```csharp
public class WebhooksWebJob
{
	public static IServiceLocator ServiceLocator { get; set; }

    public static void ProcessDropboxWebhook([QueueTrigger("dropbox-webhook")] string notification, TextWriter log)
    {
    }
}
```

And in the `Main()` method of my WebJob I hook up everything:

```csharp
class Program
{
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

        // Set up the service locator
        ServiceLocator.SetLocatorProvider(() => new AutofacServiceLocator(builder.Build()));

        // Set the service locator for the WebJob
        WebhooksWebJob.ServiceLocator = ServiceLocator.Current;

        // Start up and wait
        var host = new JobHost();
        host.RunAndBlock();
    }
}
```

And now I can change my WebJob to get an instance of `IDropboxApi` through the Service Locator:

```csharp
public class WebhooksWebJob
{
	public static IServiceLocator ServiceLocator { get; set; }

    public static void ProcessDropboxWebhook([QueueTrigger("dropbox-webhook")] string notification, TextWriter log)
    {
		IDropboxApi dropboxApi = ServiceLocator.GetInstance<IDropboxApi>();

		// Do the rest of the work...
		// ...
    }
}
```

So at this point I can easily test the interaction between my WebJob and the Dropbox API because I can mock out both `IServiceLocator` and `IDropboxApi` for unit testing purposes.

If you have a better idea on how to do this I would love to hear it, so [give me a shout on Twitter](https://twitter.com/jerriepelser).  