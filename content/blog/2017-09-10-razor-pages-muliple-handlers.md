---
date: 2017-09-10T00:00:00Z
description: |
  Razor Pages allow you to define extra handlers over and above the normal OnGet and OnPost handlers. This can be useful for handling AJAX requests.
tags:
- asp.net core
- razor pages
title: 'Razor Pages tip: Define extra handlers for a Razor Page'
url: /blog/razor-pages-muliple-handlers/
---

I have been tinkering with Razor Pages a little but, and had a requirement where I wanted to make an AJAX request to the server from a Razor Page. A Razor Page source file typically contains 2 handlers, namely `OnGet` and `OnPost` which handles `GET` and `POST` requests respectively:

```csharp
public class IndexModel : PageModel
{
    public void OnGet()
    {
        // Process GET requests 
    }

    public void OnPost()
    {
        // Process POST requests
    }
}
```

If the handler is an `async` method, you can also optionally use an `Async` suffix, for example:

```csharp
public class IndexModel : PageModel
{
    public async Task<IActionResult> OnGetAsync()
    {
        // Process GET requests 
    }

    public async Task<IActionResult> OnPostAsync()
    {
        // Process POST requests
    }
}
```

As I mentioned before, I wanted to make an AJAX request from the web page, back to the server. Well, it turns out that you can define extra handlers if you choose to. Let's say for example that I had a handler which returns a JSON result containing a list of stores: 

```csharp
public class IndexModel : PageModel
{
    public void OnGet()
    {
        // Process GET requests 
    }

    public void OnPost()
    {
        // Process POST requests
    }

    public IActionResult OnGetStores()
    {
        // return JSON result with store information
    }
}
```

You can now invoke that handler by passing along the `handler` query string parameter, and specifying the name of the handler you wish to invoke. If the name of the method is `OnGetStores`, then the name of the handler is `stores`. Same, if the name of the method was `OnGetStoresAsync`, the name of the handler would also be `stores`. 

So in other words, assuming the URL for my Page is `/map`, I can simply make a `GET` request as follows:

```bash
GET /map?handler=stores
```

I did not like passing the name of the handler in a query string, but preferred passing it as part of the path. Well it turns out you can do that as well by adding a route template to the `@page` directive:

```html
@page "{handler?}"
@model IndexModel

<!-- rest of code omitted -->
```

Now I can invoke that handler by making the following request:

```bash
GET /map/stores
```

This is fully documented, but it seems many people miss the fact that you can do this. For more info also refer to the official ASP.NET Core docs:

* [Multiple handlers per page](https://docs.microsoft.com/en-us/aspnet/core/mvc/razor-pages/?tabs=visual-studio#multiple-handlers-per-page)
* [Customizing Routing](https://docs.microsoft.com/en-us/aspnet/core/mvc/razor-pages/?tabs=visual-studio#customizing-routing)