---
date: 2015-07-28T00:00:00Z
description: |
  Demonstrates how you can generate URLs in your application which are not only more friendly to search engines, but for users as well.
tags:
- asp.net
- asp.net mvc
- seo
title: Generating SEO (and user) friendly URLs in your ASP.NET MVC Application
url: /blog/generate-seo-friendly-urls-aspnet-mvc/
---

## Introduction

Have a look at the following URL:

``` text
http://www.amazon.com/gp/product/1617292397/ref=s9_psimh_gw_p14_d4_i1?pf_rd_m=ATVPDKIKX0DER&pf_rd_s=desktop-1&pf_rd_r=0TFM5Q6WM6KD9NPNG9G7&pf_rd_t=36701&pf_rd_p=2079475242&pf_rd_i=desktop
``` 

Do you have any idea what the URL is pointing to by just looking at it? Most likely not.

Now look at this one:

``` text
http://www.dpreview.com/articles/9485436827/the-big-beast-hands-on-with-the-panasonic-lumix-dmc-gx8
```

Any idea? Well it would seem that it is an article about a hands-on with the Panasonic Lumix DMX-GX8.

In today's blog post I would like to demonstrate how to generate "friendly" URLs in your ASP.NET applications. Friendly not just in the sense that someone can look at it and figure out what the URL is pointing to, but more importantly friendly for search engines.

Right now you may probably ask what difference it makes to search engines like Google what the URL of a page is? Surely a computer does not care? Well you would be sort of right, but the thing is that having keywords you are trying to rank for in the URL of a page, does indeed make a difference. 

There are many things which play a role in how Google determines the ranking of a web page, and no one can say for certain exactly how big a part each of those factors play. What most people agree on however is that having the keywords you want a page to rank for in the URL of the page does indeed help with the ranking of the page.

In this blog post I am going to show you how to generate URLs in your application which is more SEO - and user - friendly.

## Generate URLs

In my sample application I have created a fictitious product database which displays a listing of products, and allows a user to click on a specific product to navigate to the details page for that product.

My product class is fairly simple:

``` csharp
public class Product
{
    public string Description { get; set; }
    public int Id { get; set; }
    public string Name { get; set; }
    public float Price { get; set; }
}
```

On the product listing page I simply display a list of products:

![](/assets/images/generate-seo-friendly-urls-aspnet-mvc/product-listing.png)

And when the user clicks on a specific product they are navigated to a product details page:

![](/assets/images/generate-seo-friendly-urls-aspnet-mvc/product-detail-with-id.png)

Take a look at the URL which we are generating:

![](/assets/images/generate-seo-friendly-urls-aspnet-mvc/product-id-url.png)

It is just a normal URL as you get in most ASP.NET MVC application which passes along the ID of particular database row to the controller action. 

We want to have something which is more user friendly. Something which at least also contains the name of our product. To generate a friendly URL I have created a new method on my `Product` class which generates a proper "slug" (or URL) for the product details page: 

``` csharp
public class Product
{
    public string Description { get; set; }
    public int Id { get; set; }
    public string Name { get; set; }
    public float Price { get; set; }

    // Slug generation taken from http://stackoverflow.com/questions/2920744/url-slugify-algorithm-in-c
    public string GenerateSlug()
    {
        string phrase = string.Format("{0}-{1}", Id, Name);

        string str = RemoveAccent(phrase).ToLower();
        // invalid chars           
        str = Regex.Replace(str, @"[^a-z0-9\s-]", "");
        // convert multiple spaces into one space   
        str = Regex.Replace(str, @"\s+", " ").Trim();
        // cut and trim 
        str = str.Substring(0, str.Length <= 45 ? str.Length : 45).Trim();
        str = Regex.Replace(str, @"\s", "-"); // hyphens   
        return str;
    }

    private string RemoveAccent(string text)
    {
        byte[] bytes = System.Text.Encoding.GetEncoding("Cyrillic").GetBytes(text);
        return System.Text.Encoding.ASCII.GetString(bytes);
    }
}
```

And I have also updated my listing page to use the slug as the route parameter instead of the `Id` as it did before:

``` html
<table >
    <thead>
    <tr>
        <th>Name</th>
        <th>Price</th>
        <th>Image</th>
    </tr>
    </thead>
    <tbody>
    @foreach (var product in Model)
    {
        <tr>
            <td><a href="@Url.Action("Details", "Products", new { id = @product.GenerateSlug() })">@product.Name</a></td>
            <td>@product.Price</td>
            <td><img src="http://placehold.it/100x100"/></td>
        </tr>
    }
    </tbody>
</table>
```

Now when I navigate to the product detail page I can see that we have a proper "friendly" URL which contains the name of the product: 

![](/assets/images/generate-seo-friendly-urls-aspnet-mvc/product-detail-with-slug-but-error.png)

## Parameter binding

But also notice that we have a new error. ASP.NET MVC is complaining that it was expecting an `id` route parameter, but could not find it. This is because the `Details` action on my controller is expecting an integer value for the `id` parameter: 

``` csharp
public ActionResult Details(int id)
{
	...
}
```
       
But instead of an integer, the `{id}` part of our route now contains a string. The MVC framework is trying to convert the string to an integer, but cannot do it and therefore it is passing a null value along, and then complains that the `id` parameter cannot contain a null value.

To fix this error we need to modify the `RouteData` for the route to fix up the value of the `id` route parameter.

I have created a new class called `SeoFriendlyRoute` that inherits from `Route` and have overridden the `GetRouteData` method to clean up the `id` parameter. In my implementation I call the base `GetRouteData` method and check if the route data is not null, in which case it means that I have a match for my route. 

In this case I simple check whether the `id` parameter is present and if it is I use a regular expression to extract the first part of the URL that contains the actual numerical ID, and I assign that numerical value to the `id` route value: 

```csharp
public class SeoFriendlyRoute : Route
{
    public SeoFriendlyRoute(string url, RouteValueDictionary defaults, IRouteHandler routeHandler) : base(url, defaults, routeHandler)
    {
    }

    public override RouteData GetRouteData(HttpContextBase httpContext)
    {
        var routeData = base.GetRouteData(httpContext);

        if (routeData != null)
        {
            if (routeData.Values.ContainsKey("id"))
                routeData.Values["id"] = GetIdValue(routeData.Values["id"]);
        }

        return routeData;
    }

    private object GetIdValue(object id)
    {
        if (id != null)
        {
            string idValue = id.ToString();
                
            var regex = new Regex(@"^(?<id>\d+).*$");
            var match = regex.Match(idValue);

            if (match.Success)
            {
                return match.Groups["id"].Value;
            }
        }

        return id;
    }
}
```

And the last bit is to add a new route for the path `/products/details/{id}` to use my new `SeoFriendlyRoute` route class:

``` csharp
public class RouteConfig
{
    public static void RegisterRoutes(RouteCollection routes)
    {
        routes.IgnoreRoute("{resource}.axd/{*pathInfo}");

        routes.Add("ProductDetails", new SeoFriendlyRoute("products/details/{id}",
            new RouteValueDictionary(new { controller = "Products", action = "Details" }),
            new MvcRouteHandler()));

        routes.MapRoute("Default", "{controller}/{action}/{id}", new {controller = "Home", action = "Index", id = UrlParameter.Optional}
            );
    }
}
```

And now when I refresh the page, the parameters are bound correctly as only the numeric part of the product URL that contains the actual product ID is passed along as the `id` parameter:

![](/assets/images/generate-seo-friendly-urls-aspnet-mvc/product-detail-with-slug.png)
