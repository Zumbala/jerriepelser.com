---
title: "Retry failed network requests with Polly"
description: |
  Polly is a fault-handling library which allows you to-among other things-implement a retry policy in your applications. This is ideal in situations where you have flaky network connections with intermittent connectivity issues.
date: 2018-04-02
tags:
- dotnet
- polly
url: /blog/retry-network-requests-with-polly
---

A while back I was doing work for a client, part of which involved calling an external API. I was calling this API from an Azure function and hitting it quite hard, causing it to throw intermittent errors. It turned out that these errors would typically disappear when you retry the same request a second or third time, so I decided to implement a retry pattern which will wait a few seconds, and then retry the request.

While looking for code samples I could hi-jack for implementing retry logic, I came across [Polly](https://github.com/App-vNext/Polly), which is a _fault-handling library that allows developers to express policies such as Retry, Circuit Breaker, Timeout, Bulkhead Isolation, and Fallback in a fluent and thread-safe manner._

Let's have a look at how you can implement the retry pattern in C# using Polly for scenarios like the one I described above.

## The scenario

To demonstrate the scenario, I created a simple application which will attempt to download the contents of my website and log either and informational or error message depending on whether the request was successful or not:

```csharp
class Program
{
    static async Task Main(string[] args)
    {
        var logger = new LoggerConfiguration()
            .MinimumLevel.Verbose()
            .Enrich.FromLogContext()
            .WriteTo.ColoredConsole()
            .CreateLogger();

        var httpClient = new HttpClient();
        var response = await httpClient.GetAsync("https://www.jerriepelser.com");
        
        if (response.IsSuccessStatusCode)
            logger.Information("Response was successful.");
        else
            logger.Error($"Response failed. Status code {response.StatusCode}");
    }
}
```

To simulate intermittent network errors, I have configured [Fiddler's AutoResponder](https://docs.telerik.com/fiddler/KnowledgeBase/AutoResponder) to return a 404 status code 50% of the time for requests to the `jerriepelser.com` domain:

![](/images/blog/2018-04-02-retry-network-requests-with-polly/fiddler-autoresponder.png)

This means that sometimes when I run the code above, I will get a success message:

![](/images/blog/2018-04-02-retry-network-requests-with-polly/request-success.png)

But other times I may get an error message:

![](/images/blog/2018-04-02-retry-network-requests-with-polly/request-fail.png)

## Polly to the rescue

Now, let's make this more resilient using Polly. First, install the Polly NuGet package.

```text
dotnet add package Polly
```

To implement the retry policy with Polly, we will tell it to handle an `HttpResponseMessage` result on which we will check the `IsSuccessStatusCode` property to determine whether the request was successful or not. If `IsSuccessStatusCode` is `true`, the request was successful. Otherwise, it was not.

The `WaitAndRetryAsync` method call instructs Polly to retry three times, waiting for 2 seconds between retries. We also specify an `onRetry` parameter which is a delegate that will simply log status information such as what the status code was that was returned, how long we're waiting to retry and which retry attempt this will be.

Finally, I call the `ExecuteAsync` with an `action` parameter which is a lambda that simply returns the `HttpResponseMessage` from our call to `HttpClient.GetAsync` - which Polly will pass on the handler we specified previously with the call to `HandleResult`, to determine whether the request was successful.

```csharp
static async Task Main(string[] args)
{
    var logger = new LoggerConfiguration()
        .MinimumLevel.Verbose()
        .Enrich.FromLogContext()
        .WriteTo.ColoredConsole()
        .CreateLogger();

    var httpClient = new HttpClient();
    var response = await Policy
        .HandleResult<HttpResponseMessage>(message => !message.IsSuccessStatusCode)
        .WaitAndRetryAsync(3, i => TimeSpan.FromSeconds(2), (result, timeSpan, retryCount, context) =>
        {
            logger.Warning($"Request failed with {result.Result.StatusCode}. Waiting {timeSpan} before next retry. Retry attempt {retryCount}");
        })
        .ExecuteAsync(() => httpClient.GetAsync("https://www.jerriepelser.com"));

    if (response.IsSuccessStatusCode)
        logger.Information("Response was successful.");
    else
        logger.Error($"Response failed. Status code {response.StatusCode}");
}
```

With this in place, I ran the application a few times to get to an instance where the application had to perform three retries. As you can see, Polly retried three times, waiting two seconds each time before retrying the request and finally on the third retry attempt it succeeded:

![](/images/blog/2018-04-02-retry-network-requests-with-polly/retry.png)

### Exponential back-off

In the sample above I told Polly to retry three times, and wait 2 seconds between each retry attempt, but one can also implement an exponential back-off strategy instead. For example, I can tell Polly to wait one second before the first retry, then two seconds before the second retry and finally five seconds before the last retry.

To do this, I pass an `IEnumerable<TimeSpan>` to the `WaitAndRetryAsync` method specifying the sequence of durations between retry attempts:

```csharp
var response = await Policy
    .HandleResult<HttpResponseMessage>(message => !message.IsSuccessStatusCode)
    .WaitAndRetryAsync(new[]
    {
        TimeSpan.FromSeconds(1),
        TimeSpan.FromSeconds(2),
        TimeSpan.FromSeconds(5)
    }, (result, timeSpan, retryCount, context) =>
    {
        logger.Warning($"Request failed with {result.Result.StatusCode}. Waiting {timeSpan} before next retry. Retry attempt {retryCount}");
    })
    .ExecuteAsync(() => httpClient.GetAsync("https://www.jerriepelser.com"));
```

Now when I get to a scenario where the application had to retry three times, you can see the retry policy with the exponential back-off was executed correctly:

![](/images/blog/2018-04-02-retry-network-requests-with-polly/retry-exponential.png)

## Conclusion

In this blog post I demonstrated how you can use the [Retry policy](https://github.com/App-vNext/Polly#retry) in Polly to automatically recover from errors such as intermittent network failures in your application.