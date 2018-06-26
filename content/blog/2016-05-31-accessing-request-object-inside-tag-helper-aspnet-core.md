---
date: 2016-05-31T00:00:00Z
description: |
  I look at a couple of options you can use when you want to access the Request object inside your Tag Helper.
tags:
- asp.net
- asp.net core
- taghelpers
title: Accessing the Request object inside a Tag Helper in ASP.NET Core
url: /blog/accessing-request-object-inside-tag-helper-aspnet-core/
---

Last week I was doing a little experiment which involved writing a Tag Helper. For this Tag Helper I had to access the actual URL for the request, so I therefore had to somehow get a hold of the `HttpRequest` inside of the Tag Helper.

## Injecting IHttpContextAccessor

The Request is not available as a property of the `TagHelper` base class so I figured that I needed to inject `IHttpContextAccessor` into my Tag Helper's constructor, for example:

``` csharp
public class LockTagHelper : TagHelper
{
    private readonly IHttpContextAccessor _contextAccessor;
    
    public LockTagHelper(IHttpContextAccessor contextAccessor)
    {
        _contextAccessor = contextAccessor;
    }
}
```

The Request can then later be accessed as follows:

``` csharp
var request = _contextAccessor.HttpContext.Request;
```

On my first try I got the following exception:

> InvalidOperationException: Unable to resolve service for type 'Microsoft.AspNetCore.Http.IHttpContextAccessor' while attempting to activate 'Auth0.AspNetCore.Mvc.TagHelpers.LockTagHelper'.

I know this worked before when I used ASP.NET Core (then still called ASP.NET 5) last year, and after a bit of research it seemed that the default behaviour has changed and you now had to configure `IHttpContextAccessor` manually with the DI framework. 

So inside the `ConfigureServices` method of your `Startup` class, simple add the following line.

``` csharp
services.AddSingleton<IHttpContextAccessor, HttpContextAccessor>();
```

This worked great but it posed a problem for me. This particular Tag Helper would be available as a NuGet package and I did not want to expect users to have to configure `IHttpContextAccessor` with the DI in order for my Tag Helper to work correctly.

## Using ViewContextAttribute

I needed a way which was less error prone, and after [posing the question on GitHub](https://github.com/aspnet/Mvc/issues/4744), [Pranav](https://twitter.com/pranav_km) supplied a much better solution..

Simply declare a property of type `ViewContext` and decorate it with the `[ViewContext]` attribute.

You can then access the `HttpRequest` through the `ViewContext.HttpContext.Request` property. 

``` csharp
public class LockTagHelper : TagHelper
{
    protected HttpRequest Request => ViewContext.HttpContext.Request;
    protected HttpResponse Response => ViewContext.HttpContext.Response;

    [ViewContext]
    public ViewContext ViewContext { get; set; }

    // Code omitted for brevity
}
```
