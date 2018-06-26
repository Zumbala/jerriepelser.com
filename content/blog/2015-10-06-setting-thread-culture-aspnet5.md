---
date: 2015-10-06T00:00:00Z
description: |
  A quick note on setting the Culture in your ASP.NET 5 application to a specific setting.
tags:
- asp.net
- asp.net 5
- asp.net mvc
- localization
title: Setting the thread culture in ASP.NET 5
url: /blog/setting-thread-culture-aspnet5/
---

## Introduction

I recently had to figure out how to set the Culture in an ASP.NET 5 application to a specific setting. The documentation on this is still non-existing, so this is how I figured out doing it by looking at Damian Edwards' [i18nStarterWeb](https://github.com/DamianEdwards/i18nStarterWeb) sample project on Github.

This information is current as of Beta 7. It may change in the future.

## The ASP.NET 4.6 way
  
In ASP.NET 4.6 and before there were two typical ways in which you would be able to set the Culture and UI Culture to a specific value for a website. Let us say for example that I wanted to force a website to use the culture settings for Afrikaans (my own native language).  

The first way I could achieve this was to do it declaratively in the `web.config` by adding a `<globalization>` element:

``` xml
<system.web>
	<globalization uiCulture="af" culture="af" />
</system.web>
```

As second possible way was to specify a handler in the `Global.asax` file which would set the Culture and UI Culture for each request:

```csharp
protected void Application_BeginRequest()
{
    Thread.CurrentThread.CurrentCulture = new CultureInfo("af");
    Thread.CurrentThread.CurrentUICulture = new CultureInfo("af");
}
```

## The ASP.NET 5 way

In ASP.NET 5 the way you would do this has changed. If I wanted to force the website to use the culture settings for Afrikaans, I would have to go to the `Startup.cs` file and in the `Configure()` method use the `UseRequestLocalization()` extension method to configure the appropriate settings.

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    loggerFactory.MinimumLevel = LogLevel.Information;
    loggerFactory.AddConsole();
    loggerFactory.AddDebug();

	// Configure the localization options
    app.UseRequestLocalization(new RequestLocalizationOptions
    {
        DefaultRequestCulture = new RequestCulture(new CultureInfo("af"))
        SupportedCultures = new List<CultureInfo>
        {
            new CultureInfo("af")
        },
        SupportedUICultures = new List<CultureInfo>
        {
            new CultureInfo("af")
        }
    });

	// rest of the method omitted for brevity
	// ...
}
```

You need to set the `DefaultRequestCulture` property to force the specific culture to be the default one, but note that you also need to specify the culture as being one of the available cultures by adding it to both the `SupportedCultures` and to the `SupportedUICultures` lists.

Hope this helps! If there is another, more correct way of doing this then please leave a comment down below. 
