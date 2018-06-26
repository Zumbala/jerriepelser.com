---
date: 2015-10-20T00:00:00Z
description: |
  I describe the steps the ASP.NET 5 Localization middleware takes to determine the correct culture settings for a specific request.
tags:
- asp.net
- asp.net 5
- asp.net mvc
- localization
title: How ASP.NET 5 determines the culture settings for localization
url: /blog/how-aspnet5-determines-culture-info-for-localization/
---

## Introduction

In ASP.NET 5, the Localization middleware can be used to ensure the correct culture settings are applied when serving a request. In this blog post I will walk you through the process which the Localization middleware takes to determine the correct culture settings to use when serving a request.

I am using Beta 8, and have created a simple web application using the [ASP.NET 5 Yeoman generator](https://www.npmjs.com/package/generator-aspnet).  

## Configuring the Localization middleware

The first thing you will need to do is to enable the Localization middleware inside your application. For that you can head over to the `Configure` method in the `Startup.cs` file and add a call to the `UseRequestLocalization()` method. 

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    loggerFactory.MinimumLevel = LogLevel.Information;
    loggerFactory.AddConsole();
    loggerFactory.AddDebug();

    // Configure the Localization middleware
    app.UseRequestLocalization();

    // Rest of method omitted for brevity...
}
```

Enabling the middleware is as simple as that. But let us look at how it works...

## How the Localization middleware sets the culture

To understand how the Localization middleware works we can look at the [source code on GitHub](https://github.com/aspnet/Localization), specifically the `Invoke()` method of the `RequestLocalizationMiddleware` class:

```csharp
public async Task Invoke(HttpContext context)
{
    if (context == null)
    {
        throw new ArgumentNullException(nameof(context));
    }

    var requestCulture = _options.DefaultRequestCulture ??
        new RequestCulture(CultureInfo.CurrentCulture, CultureInfo.CurrentUICulture);

    IRequestCultureProvider winningProvider = null;

    if (_options.RequestCultureProviders != null)
    {
        foreach (var provider in _options.RequestCultureProviders)
        {
            var result = await provider.DetermineRequestCulture(context);
            if (result != null)
            {
                requestCulture = result;
                winningProvider = provider;
                break;
            }
        }
    }

    context.Features.Set<IRequestCultureFeature>(new RequestCultureFeature(requestCulture, winningProvider));

    SetCurrentThreadCulture(requestCulture);

    await _next(context);
}
```

The middleware first determines the default (or fallback) culture if all other methods fail. By default this is the culture settings of the current thread, but you can override that by specifying a default culture in the `RequestLocalizationOptions` which you can optionally pass to the `UseRequestLocalization()` method (we will look at how to do this shortly).

> For more information on how the culture info for a thread gets determined, you can refer to the "How a Thread's Culture Is Determined" section in the MSDN documentation for the [CultureInfo.CurrentCulture property](https://msdn.microsoft.com/en-us/library/system.globalization.cultureinfo.currentculture%28v=vs.110%29.aspx).
 
Then the middleware goes through the list of Request Culture Providers (which can also be overridden in the `RequestLocalizationOptions`), and see whether any of them can determine the culture. If any of them can, then it uses that as the culture. If none of them can, then if uses the default culture which it determined at the beginning of the method. 

Now you may wonder what the possible Request Culture Providers are. For that we can head to the [source code for the `RequestLocalizationOptions` class](https://github.com/aspnet/Localization/blob/dev/src/Microsoft.AspNet.Localization/RequestLocalizationOptions.cs), and in the construcutor we can see the possble providers being specified.

```csharp
public RequestLocalizationOptions()
{
    DefaultRequestCulture = new RequestCulture(CultureInfo.CurrentCulture, CultureInfo.CurrentUICulture);

    RequestCultureProviders = new List<IRequestCultureProvider>
    {
        new QueryStringRequestCultureProvider { Options = this },
        new CookieRequestCultureProvider { Options = this },
        new AcceptLanguageHeaderRequestCultureProvider { Options = this }
    };
} 
```

So with that knowledge, here is the order in which the correct culture gets determined (in order of priority):

1. From the query string
2. From a cookie
3. From the Accept-Language HTTP header
4. From the `DefaultRequestCulture` property of the `RequestLocalizationOptions` class
5. From the thread culture

So the query string always takes the highest priority, then the cookie, then the Accept-Language header, etc..

## Using each of the methods

This would not be a complete blog post if I stopped right there. We need to look at how you can use each of these in your application, and for that I am going to start with the lowest priority one, namely the thread culture, and work my way up to show you the effect of each.

I have changed the source code for my `Index.cshtml` file to simply display the Current Culture and Current UI Culture:

```html
@using System.Globalization
@{
    ViewData["Title"] = "Home Page";
}

<div class="container">
    <div class="row">
        <div class="col-md-12">
            <div>Current Culture: @CultureInfo.CurrentCulture.DisplayName</div>
            <div>Current UI Culture: @CultureInfo.CurrentUICulture.DisplayName</div>
        </div>
    </div>
</div>
```

So lets look at each of those methods in more detail.   

### Thread Culture

As stated before, this is the default option when all else fails. The thing is however, that I cannot really demonstrate this in a browser, because the browser is going send an `Accept-Language` header, and I do not what that to override the default behaviour. So to demonstrate this correctly, I am going to use the CURL command from Windows PowerShell and display the raw content retrieved:

![](/assets/images/how-aspnet5-determines-culture-info-for-localization/powershell-curl.png)  

When I run the command I can view the raw HTML:

![](/assets/images/how-aspnet5-determines-culture-info-for-localization/powershell-curl-default.png)

As you can see the culture is set to "English (South Africa)" and the UI Culture is set to "English (United States)". This is based on my user's culture settings in Windows.

### Setting the default request culture  

The first way we can override the culture settings is by setting a culture using the `DefaultRequestCulture` property of the `RequestLocalizationOptions`. So let's change the call to `UseRequestLocalization` and set "en-US" as the default culture:

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    loggerFactory.MinimumLevel = LogLevel.Information;
    loggerFactory.AddConsole();
    loggerFactory.AddDebug();

    // Configure the Localization middleware
    app.UseRequestLocalization(new RequestLocalizationOptions
    {
        DefaultRequestCulture = new RequestCulture(new CultureInfo("en-US"))
    });

    // Rest of method omitted for brevity...
}
```

Now doing the PowerShell CURL magic, we can see that both the Culture and UI Culture is set to "English (United States)".

![](/assets/images/how-aspnet5-determines-culture-info-for-localization/powershell-curl-default-override.png) 

### The Accept-Language HTTP header

Next up is the Accept-Language HTTP Header, so for this one we can head over to the browser. I am using Google Chrome and this is what my Google Chrome Language settings looks like:

![](/assets/images/how-aspnet5-determines-culture-info-for-localization/google-chrome-language-settings.png)

And if I go to the web page in my browser, you will see that Chrome sends the language settings in the Accept-Language HTTP header, and you can see the resulting page display "English (United Kingdom)" as the culture: 

![](/assets/images/how-aspnet5-determines-culture-info-for-localization/accept-language-result.png)

### Setting the cookie

The Localization middleware can also look for a cookie which is set. The name of the cookie it is looking for is `ASPNET_CULTURE` and you can pass the values for the Culture and UI Culture using "c=<culture>" and "uic=<culture>" and separating them with a pipe character. So to set the culture to Afrikaans (af-ZA), I can set the value for the cookie to:

```text
c=af-ZA|uic=af-ZA
```

> Note that it is important that you set *both* the culture and the UI culture. You can set them to different cultures but you *must* set a value for both, otherwise the culture will not be overridden. This is currently the case in Beta 8, though that may change in the future.

So I use a Chrome Plugin called [EditThisCookie](http://www.editthiscookie.com/) to manually set the cookie value:

![](/assets/images/how-aspnet5-determines-culture-info-for-localization/setting-cookie-value.png)

And when I refresh the page you can see that both the culture and UI culture is set to "Afrikaans (South Africa)".

![](/assets/images/how-aspnet5-determines-culture-info-for-localization/culture-afrikaans.png)

### Passing the culture in the query string

The last way to override the culture is to pass it in a query string. The query string parameter you should pass for the culture is called "culture" and for the UI Culture it is called "ui-culture". So to specify Mexican Spanish I can pass in the following URL:

``` text
http://localhost:5000/?culture=es-MX&ui-culture=es-MX
```

And that will result in the following:

![](/assets/images/how-aspnet5-determines-culture-info-for-localization/culture-query-string.png)

Note that with the query provider you can pass in just the `culture` or `ui-culture` which will set both values to whatever you pass in. So passing just the culture will set both the Culture and the UI Culture:

![](/assets/images/how-aspnet5-determines-culture-info-for-localization/culture-query-string2.png)

It is strange to me because this behaviour is different from the Cookie provider which requires you to set *both* values. Maybe this is intentional, maybe it is not, but I find the inconsistency confusing. I have [logged an issue](https://github.com/aspnet/Localization/issues/116) on GitHub regarding this, so hopefully they make it consistent across both these providers. 

## Conclusion

In this blog post I showed you how the Localization middleware in ASP.NET 5 determines the correct Culture for a request.

The 3 built-in providers are the Accept-Language Header provider, the Cookie provider and the Query String provider. The model is completely extensible so you can write your own provider to set the culture. Maybe your users can specify the culture in their profile, so you can write a provider to set it based on the user's profile.

You are also free to swap around the order of the providers, so that the Cookie provider for example always takes highest priority. You can even remove some of the providers altogether. It's up to you how you want to handle this. What I demonstrated is the default behaviour.
