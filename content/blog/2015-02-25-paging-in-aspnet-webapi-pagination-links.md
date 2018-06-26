---
date: 2015-02-25T00:00:00Z
description: |
  Looking at how we can allow users of our API to page through result sets by giving them pagination links to follow when requesting.
tags:
- aspnet
- aspnet mvc
- aspnet web api
title: 'Paging in ASP.NET Web API: Generating Pagination Links'
url: /blog/paging-in-aspnet-webapi-pagination-links/
---

## Introduction

So far in the series we have looked in the first post at an [introduction to paging in REST APIs](http://www.jerriepelser.com/blog/paging-in-aspnet-webapi-introduction) and then looked at some specific implementations in ASP.NET Web API. In the second post I demonstrated how you could [return pagination information in a simple JSON envelope](http://www.jerriepelser.com/blog/paging-in-aspnet-webapi-json-envelope), and in the third post I [returned and requested all pagination information through HTTP headers](http://www.jerriepelser.com/blog/paging-in-aspnet-webapi-http-headers). One constant so far in the previous two posts was that we would return details such as the page number, page size and number of pages in the pagination information. 

There is however a different approach which APIs [like GitHub](https://developer.github.com/v3/#pagination) takes and that is to return links which allow the end user of the API to page between different result sets. This means that the user of the API don't have to build up the URL to do paging, we give it to them. In the first part of this blog post I will show how to build up the links and return them as part of a JSON envelope. In the second part we will return them in the [HTTP Link header](http://tools.ietf.org/html/rfc5988).

So, let's get coding... :) 

## Building a link builder helper class

I am going to build up a little helper class which builds up the pagination links, but I first need to give a quick introduction to named routes in Web API. When using the `RouteAttribute` in Web API for specifying routes, you can pass in an extra parameter to specify the name of the route. This is very useful when you want to create a URL to that route. So let us say for example we have the following API method with the named route "GetPageLinks".

``` csharp
[HttpGet]
[Route("customers/pagelinks", Name = "GetPageLinks")]
public IHttpActionResult GetPageLinks(int pageNo = 1, int pageSize = 50)
{
	...
}  
```

If at some point I want to generate a URL to that specific method I could use the `Url` property (an instance of the `UrlHelper` class) on the `ApiController` base class to generate a URL. For example,

``` csharp
Url.Link("GetPageLinks", new { pageNo = 1, pageSize = 50 });
```

Will generate the URL

``` text
http://localhost:7965/api/customers/pagelinks?pageNo=1&pageSize=50
```

So with all that, here is the code for the `PageLinkBuilder` helper class:

``` csharp
public class PageLinkBuilder
{
    public Uri FirstPage { get; private set; }
    public Uri LastPage { get; private set; }
    public Uri NextPage { get; private set; }
    public Uri PreviousPage { get; private set; }

    public PageLinkBuilder(UrlHelper urlHelper, string routeName, object routeValues, int pageNo, int pageSize, long totalRecordCount)
    {
        // Determine total number of pages
        var pageCount = totalRecordCount > 0
            ? (int) Math.Ceiling(totalRecordCount/(double) pageSize)
            : 0;

        // Create them page links 
        FirstPage = new Uri(urlHelper.Link(routeName, new HttpRouteValueDictionary(routeValues)
        {
            {"pageNo", 1},
            {"pageSize", pageSize}
        }));
        LastPage = new Uri(urlHelper.Link(routeName, new HttpRouteValueDictionary(routeValues)
        {
            {"pageNo", pageCount},
            {"pageSize", pageSize}
        }));
        if (pageNo > 1)
        {
            PreviousPage = new Uri(urlHelper.Link(routeName, new HttpRouteValueDictionary(routeValues)
            {
                {"pageNo", pageNo - 1},
                {"pageSize", pageSize}
            }));
        }
        if (pageNo < pageCount)
        {
            NextPage = new Uri(urlHelper.Link(routeName, new HttpRouteValueDictionary(routeValues)
            {
                {"pageNo", pageNo + 1},
                {"pageSize", pageSize}
            }));
        }
    }
}
```

And to call it from a method in our API controller we simply call it like this:

``` csharp
var linkBuilder = new PageLinkBuilder(Url, "GetPageLinks", null, pageNo, pageSize, total);
```

You will notice that in the `PageLinkBuilder` I don't pass an anonymous object to the `Link` method of the `UrlHelper` class (like I did in my overview above), but instead pass an instance of `HttpRouteValueDictionary`. This is just to allow a little bit more flexibility in the number of parameters which I can pass in to `PageLinkBuilder` class. You will see the third parameter is called `routeValues` which you can use to pass in extra parameter values, over and above the page number and page size parameters. 

I then create a `HttpRouteValueDictionary` which is initialised from the `routeValues` parameter, and I add the `pageNo` and `pageSize` parameters to the dictionary. So if my "GetPageLinks" method also took a "sort" parameter for example, I could call it like this:

``` csharp
var linkBuilder = new PageLinkBuilder(Url, "GetPageLinks", new { sort = "Firstname" }, pageNo, pageSize, total);
```

and that would then potentially generate the link

``` text
http://localhost:7965/api/customers/pagelinks?sort=FirstName&pageNo=1&pageSize=50
```

## Returning pagination links in a JSON envelope

With all that behind us I can simply copy the code I used in my blog post on returning a [JSON envelope](http://www.jerriepelser.com/blog/paging-in-aspnet-webapi-json-envelope) and modify it a little bit to call the `PageLinkBuilder` class. I then take the links returned from the `PageLinkBuilder` and use that to build up a JSON envelope which contains the list of customers in a `Data` property and all the pagination links in a `Paging` property. 

``` csharp
[HttpGet]
[Route("customers/pagelinks", Name = "GetPageLinks")]
public IHttpActionResult GetPageLinks(int pageNo = 1, int pageSize = 50)
{
    // Determine the number of records to skip
    int skip = (pageNo - 1) * pageSize;

    // Get total number of records
    int total = _dbContext.Customers.Count();

    // Select the customers based on paging parameters
    var customers = _dbContext.Customers
        .OrderBy(c => c.Id)
        .Skip(skip)
        .Take(pageSize)
        .ToList();

    // Get the page links
    var linkBuilder = new PageLinkBuilder(Url, "GetPageLinks", null, pageNo, pageSize, total);

    // Return the list of customers
    return Ok(new
    {
        Data = customers,
        Paging = new
        {
            First = linkBuilder.FirstPage,
            Previous= linkBuilder.PreviousPage,
            Next = linkBuilder.NextPage,
            Last = linkBuilder.LastPage
        }
    });
}
```
And this is what the data returned from the API looks like:

![](/assets/images/2015-02-25-paging-in-aspnet-webapi-pagination-links/postman-pagination-links-envelope.png)

> If you have not followed my previous posts, just a note that I use [Postman](http://www.getpostman.com/) to test the API as shown in the screenshot above. 

## Return them in the Link Header

As I said in the introductory paragraph, I will also show you how to return the pagination links in the HTTP Link header. You can go and read the [HTTP Link header](http://tools.ietf.org/html/rfc5988) specification, but it is pretty much impossible for any normal human being to understand that. So herewith the short version of what we want to achieve:

The link header contains multiple values which are separated by a comma (,). Each of that values contains a URI inside angle brackets (e.g. <URI>), as well as a "rel" parameter which specifies relation type of the link. In our case it will depend on where the URI points to, e.g. "first" for the URI to the first page, "next" for the URI to the next page etc.

So a single link value will look like this:

``` text
<http://localhost:7965/api/customers/pagelinks?pageNo=1&pageSize=50>; rel="first"
```

And adding a bunch of them together, the Link header will look like this:

```text
Link: <http://localhost:7965/api/customers/pagelinks?pageNo=1&pageSize=50>; rel="first",
	  <http://localhost:7965/api/customers/pagelinks?pageNo=2&pageSize=50>; rel="prev",
      <http://localhost:7965/api/customers/pagelinks?pageNo=4&pageSize=50>; rel="next",
      <http://localhost:7965/api/customers/pagelinks?pageNo=5&pageSize=50>; rel="last"
``` 

> Linebreaks in example header above included for readability :)

So same as per the JSON envelope example above we will create an instance of the `PageLinkBuilder` class to help construct the links, and then use that to build up the links. Also once again, same as in the previous post on [returning pagination information via the HTTP headers](http://www.jerriepelser.com/blog/paging-in-aspnet-webapi-http-headers), I will opt for using a `HttpResponseMessage` as the result of my API method, instead of using `IHttpActionResult` as it makes it easier to manipulate the headers. 

Here's the code for the full method:

``` csharp
[HttpGet]
[Route("customers/pagelinkheaders", Name = "GetPageLinkHeaders")]
public HttpResponseMessage GetPageLinkHeaders(int pageNo = 1, int pageSize = 50)
{
    // Determine the number of records to skip
    int skip = (pageNo - 1) * pageSize;

    // Get total number of records
    int total = _dbContext.Customers.Count();

    // Select the customers based on paging parameters
    var customers = _dbContext.Customers
        .OrderBy(c => c.Id)
        .Skip(skip)
        .Take(pageSize)
        .ToList();

    // Get the page links
    var linkBuilder = new PageLinkBuilder(Url, "GetPageLinkHeaders", null, pageNo, pageSize, total);

    // Create the response
    var response = Request.CreateResponse(HttpStatusCode.OK, customers);

    // Build up the link header
    List<string> links = new List<string>();
    if (linkBuilder.FirstPage != null)
        links.Add(string.Format(LinkHeaderTemplate, linkBuilder.FirstPage, "first"));
    if (linkBuilder.PreviousPage != null)
        links.Add(string.Format(LinkHeaderTemplate, linkBuilder.PreviousPage, "previous"));
    if (linkBuilder.NextPage != null)
        links.Add(string.Format(LinkHeaderTemplate, linkBuilder.NextPage, "next"));
    if (linkBuilder.LastPage != null)
        links.Add(string.Format(LinkHeaderTemplate, linkBuilder.LastPage, "last"));

    // Set the page link header
    response.Headers.Add("Link", string.Join(", ", links));

    // Return the response
    return response;
}
```

And the result in Postman:

![](/assets/images/2015-02-25-paging-in-aspnet-webapi-pagination-links/postman-pagination-links-header.png)

## Conclusion

In this post I showed you how to return pagination links which contains URIs to navigate between pages, instead of returning the page number, page size etc. I also showed how to return the links inside a JSON envelope, as well as in the HTTP Link header. I think this is about as much as I am going to do in this series with simple pagination over fairly static data sets like customers. 

You should now have quite a number of options which you can use when adding pagination to your own APIs. There really is no "standard" way to do something like this. Many different APIs utilise many different ways to do pagination. In the end it is your API so you need to choose something which makes sense for you and the users of your API.

Next up we will look at pagination over more dynamic, constantly changing, data sets like a Twitter timeline.

By the way, if you are curious on how one would build up pagination links at the bottom of your website when using this method, check out the blog post entitled [Traversing with Pagination](https://developer.github.com/guides/traversing-with-pagination/) from the guys at GitHub.