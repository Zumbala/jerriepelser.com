---
date: 2017-05-18T00:00:00Z
description: |
  You can determine a user's location from their IP Address. It is not as accurate as the HTML 5 Geolocation API, but their are many instances where this is acceptable.
tags:
- .net core
- asp.net
- asp.net core
- maxmind
title: Determine a user's location from their IP Address in ASP.NET Core
url: /blog/aspnetcore-geo-location-from-ip-address/
---

To determine a user's location in a web browser you can use the [HTML Geolocation API](https://www.w3schools.com/html/html5_geolocation.asp). There are however a couple of issues with this.

First, the user will be prompted to give permission for you to determine their location. If they deny this request, then you will not be able to determine the location.

![](/assets/images/2017-05-18-aspnetcore-geo-location-from-ip-address/location-permission-prompt.png)

Secondly, this is a JavaScript API, and therefore a client-side solution.

But what if you want to determine a user's location on the server? One way in which you can do this is by determining the user's location from their IP Address (also referred to as Geo IP). 

Even though this method is not as accurate as the HTML Geolocation API (the level of accuracy is typically limited to the city the user is in), there are many instances where this is acceptable, for example:

* You only want to make content available to users in a specific country.
* You want to determine a user's time zone - which you can infer from their city.
* Maybe you are a nationwide chain store, and have a special promotion you want to display to users in a specific city.
* The list goes on...

## Getting a database of IP Addesses

If you want to implement this functionality, the first thing you need to do is to find a database or some sort of service which will do a Geolocation lookup from an IP Address. There are various of these available, but the one I am going to focus on in this blog post is [MaxMind](https://www.maxmind.com). They have both a downloadable database and a web service you can use.

In this blog post I will be using their [Geolite2 database](https://www.maxmind.com/en/geolite2-developer-package) which is free, but not as accurate as their paid offering.

They have various SDKs available and .NET developers can use their [GeoIP2 .NET API](https://github.com/maxmind/GeoIP2-dotnet). ASP.NET Core developers are in luck as the NuGet package supports .NET Standard 1.4.

## Using the GeoIP2 .NET API

To demonstrate how to implement this, I have a created a standard ASP.NET Core MVC application using the project templates in Visual Studio 2017.

First thing you need to do is to add the GeoIP2 .NET API NuGet package:

```text
Install-Package MaxMind.GeoIP2
```

Next, you'll need to [download the MaxMind Geolite database](http://dev.maxmind.com/geoip/geoip2/geolite2/). They have various ones on offer. Go ahead and grab the **GeoLite2 City** one, and once you have done this, extract the ZIP file and copy the `.mmdb` file to your ASP.NET project folder.

Next, all you need to do is to write some code which will open a `DatabaseReader` instance, and then call the `City()` method, passing the IP Address you want to lookup. In an ASP.NET Core application, the IP Address can be determined by using `HttpContext.Connection.RemoteIpAddress`.

So with that, here is the full code for my controller action:

```csharp
public class HomeController : Controller
{
    private readonly IHostingEnvironment _hostingEnvironment;

    public HomeController(IHostingEnvironment hostingEnvironment)
    {
        _hostingEnvironment = hostingEnvironment;
    }

    public IActionResult Index()
    {
        using (var reader = new DatabaseReader(_hostingEnvironment.ContentRootPath + "\\GeoLite2-City.mmdb"))
        {
            // Determine the IP Address of the request
            var ipAddress = HttpContext.Connection.RemoteIpAddress;

            // Get the city from the IP Address
            var city = reader.City(ipAddress);

            return View(city);
        }
    }
}
```

And the Razor code for my view:

```html
@model MaxMind.GeoIP2.Responses.CityResponse
@{
    ViewData["Title"] = "Home Page";
}

<div class="row" style="margin-top: 50px;">
    <div class="col-md-12">
        <p>Hey there, welcome visitor from <strong>@Model.City, @Model.Country!</strong></p>
        <p>Your time zone is <strong>@Model.Location.TimeZone</strong></p>
    </div>
</div>
```

## Testing on your local machine

When you try and test this on your local machine, your IP Address will resolve to the loopback address (i.e. `127.0.0.1` or `::1`). A handy way in which you can fool ASP.NET Core in thinking the request is coming from somewhere else is by using the [ForwardedHeadersMiddleware](https://github.com/aspnet/BasicMiddleware/blob/rel/1.1.1/src/Microsoft.AspNetCore.HttpOverrides/ForwardedHeadersMiddleware.cs) and passing along a [X-Forwarded-For](https://en.wikipedia.org/wiki/X-Forwarded-For) header with each request.

First, register the `ForwardedHeadersMiddleware` when running in Development mode by calling the `UseForwardedHeaders` extension method. You can pass along an instance of `ForwardedHeadersOptions` for which your need to set the `ForwardedHeaders` to look only for the `X-Forwarded-For`. 

When running on IIS, you will also need to set the `ForwardLimit` to **2**. By default this is set to **1**, but IIS already acts as a reverse proxy and will add a `X-Forwarded-For` header to all requests. If the `ForwardLimit` is set to **1**, then the middleware will only pick up the value which was set by IIS, and not the value you are passing in. So be sure to set `ForwardLimit` to **2** - it had me scratching my head for a while.

```csharp
public class Startup
{
    // Some code omitted from this code sample for brevity...

    public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
    {
        if (env.IsDevelopment())
        {
            app.UseForwardedHeaders(new ForwardedHeadersOptions
            {
                ForwardedHeaders = ForwardedHeaders.XForwardedFor,

                // IIS is also tagging a X-Forwarded-For header on, so we need to increase this limit, 
                // otherwise the X-Forwarded-For we are passing along from the browser will be ignored
                ForwardLimit = 2 
            });

            app.UseDeveloperExceptionPage();
            app.UseBrowserLink();
        }
        else
        {
            app.UseExceptionHandler("/Home/Error");
        }

        app.UseStaticFiles();

        app.UseMvc(routes =>
        {
            routes.MapRoute(
                name: "default",
                template: "{controller=Home}/{action=Index}/{id?}");
        });
    }
}
```

The last thing we need to do is to be able to add the `X-Forwarded-For` header to all requests. If you are using Chrome you can install the [ModHeader extension](https://chrome.google.com/webstore/detail/modheader/idgpnmonknjnojddfkpgkljpfnnfcklj) which will allow you to modify the HTTP headers of requests.

## Testing it out 

Great, now we can test this out! We will however need a IP Address to test with. The easiest way is to type "what is my ip" in Google, and it will tell you:

![](/assets/images/2017-05-18-aspnetcore-geo-location-from-ip-address/what-is-my-ip.png)

Now add an `X-Forwarded-For` header in ModHeader with that IP Address:

![](/assets/images/2017-05-18-aspnetcore-geo-location-from-ip-address/mod-header-ip.png)

And when I run my application, I get the following output:

![](/assets/images/2017-05-18-aspnetcore-geo-location-from-ip-address/location-bangkok.png)

If you want some other ones to try out you can head over to the [IP Address Ranges by Country](http://lite.ip2location.com/ip-address-ranges-by-country) page, which will give you all the IP Address ranges assigned to a specific country. It won't give it to you on city level, but it is good enough to test a few others out.

As you can see when I tested one of the IP addresses from Afghanistan:

![](/assets/images/2017-05-18-aspnetcore-geo-location-from-ip-address/location-kabul.png)

## More info

Check out the [GeoIP2 .NET API documentation](http://maxmind.github.io/GeoIP2-dotnet/) for more information on how to use it in your application.

Source code for this sample is available at [https://github.com/jerriepelser-blog/aspnetcore-geo-location-from-ip](https://github.com/jerriepelser-blog/aspnetcore-geo-location-from-ip). **Please note** that I have not checked the actual MaxMind database into GitHub as it is huge. If you want to play with my sample you will have to download it yourself, as I described previously in this blog post.