---
date: 2015-02-18T00:00:00Z
description: |
  Looking at how we can implement paging in ASP.NET Web API by sending paging meta data in HTTP Headers. Also looks how you can request specific pages through headers.
tags:
- aspnet
- aspnet mvc
- aspnet web api
title: "Paging in ASP.NET Web API: Using HTTP Headers"
url: /blog/paging-in-aspnet-webapi-http-headers/
---

## Introduction

In the [previous post](http://www.jerriepelser.com/blog/paging-in-aspnet-webapi-json-envelope) I did paging using both an offset based mechanism as well as a page based mechanism and returned the result inside a JSON envelope. In this blog post I will discard the envelope and return a simple JSON array of objects while returning the paging meta information in HTTP Headers. In the second part of the blog post I will also change the Web API method so that users of the API can request a specific page and specify the page size through a HTTP headers instead of query string parameters.

## Return paging information in HTTP response headers

Getting rid of the JSON envelope means that we will simply return a normal JSON array which contains the list of records. The returned JSON payload will therefore look something like this: 
 
``` json
[
  {
    "firstName": "Gerard",
    "id": 1,
    "lastName": "Willms"
  },
  {
    "firstName": "Jermaine",
    "id": 2,
    "lastName": "Koelpin"
  },
  ...
  {
    "firstName": "Tevin",
    "id": 50,
    "lastName": "Robel"
  }
]
```

To achieve this we use the exact same code we used in the previous blog post, but instead of returning the a `PagedResult` we will simply return the customers list.  

``` csharp
[HttpGet]
[Route("customers/headers")]
public IHttpActionResult GetViaHeaders(int pageNo = 1, int pageSize = 50)
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

    // Return the list of customers
    return Ok(customers);
}
```

Calling this API method in [Postman](http://www.getpostman.com/), we will see the simple JSON array returned without the envelope:

![](/assets/images/2015-02-18-paging-in-aspnet-webapi-http-headers/postman-json-array.png)

For the headers we want to return the same data we previously returned in the envelope, so in other words the page number, page size, number of pages and the total number of records. We already have the page number, page size and total number of records. We add a bit of simple arithmetic to calculate the number of pages, and then return the information in the headers. 

At this point we are also changing the return type of the method from `IHttpActionResult` to `HttpResponseMessage` as it makes it easier to manipulate the headers of the response (Thank you [Filip W](https://twitter.com/filip_woj) for pointing me in the right direction with this one!). Once we have created the reponse message we simply add the relevant headers using the `HttpResponseMessage.Headers.Add()` method and finally return the response:

``` csharp
[HttpGet]
[Route("customers/headers")]
public HttpResponseMessage GetViaHeaders(int pageNo = 1, int pageSize = 50)
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

    // Determine page count
    int pageCount = total > 0
        ? (int) Math.Ceiling(total/(double) pageSize)
        : 0;

    // Create the response
    var response = Request.CreateResponse(HttpStatusCode.OK, customers);

    // Set headers for paging
    response.Headers.Add("X-Paging-PageNo", pageNo.ToString());
    response.Headers.Add("X-Paging-PageSize", pageSize.ToString());
    response.Headers.Add("X-Paging-PageCount", pageCount.ToString());
    response.Headers.Add("X-Paging-TotalRecordCount", total.ToString());

    // Return the response
    return response;
}
```

We call the API method again in Postman, and upon inspecting the **Headers** tab you will notice the paging information returned in the response headers: 

![](/assets/images/2015-02-18-paging-in-aspnet-webapi-http-headers/postman-response-headers.png)
 
## Requesting page via HTTP request headers

Great, now that we are returning the paging information for the response in the HTTP Headers, let us turn our attention to also requesting a specific page and specifying the page size with the request's HTTP headers. The current method signature for our method looks like this:

``` csharp
[HttpGet]
[Route("customers/headers")]
public HttpResponseMessage GetViaHeaders(int pageNo = 1, int pageSize = 50)
{
	...
}
```

Notice the `pageNo` and `pageSize` parameters. Those are currently bound to query string parameters. So in other word if we want to request the 2nd page and specify a page size of 20 records we will make the following request:

``` text
GET customers/headers?pageNo=2&pageSize=20
```

Binding those parameters to HTTP headers instead can be achieved by writing a custom Value Provider. For more detail on Value Providers you can refer to the relevant section in the blog post entitled [Parameter Binding in ASP.NET Web API](http://www.asp.net/web-api/overview/formats-and-model-binding/parameter-binding-in-aspnet-web-api) on the ASP.NET website which was written by Mike Wasson. In the example in his blog post he demonstrates how to write a provider which binds to a cookie. 

Doing a Google search I found an example from the book [Pro ASP.NET Web API: HTTP Web Services in ASP.NET](http://www.amazon.com/gp/search?index=books&linkCode=qs&keywords=9781430247265) which implements a Value Provider called `XHeaderValueProvider` which binds to an HTTP header with an "X-" prefix. So for example, the value of the HTTP header named "X-PageNo" will be bound to the parameter named `pageNo`. This is exactly what we are looking for, so herewith the code from the `XHeaderValueProvider` and `XHeaderValueProviderFactory` classes:

``` csharp
public class XHeaderValueProvider : IValueProvider
{
    private readonly HttpRequestHeaders _headers;
    private const string XHeaderPrefix = "X-";

    public XHeaderValueProvider(HttpActionContext actionContext)
    {
        _headers = actionContext.ControllerContext.Request.Headers;
    }
    public bool ContainsPrefix(string prefix)
    {
        return _headers.Any(header => header.Key.Contains(XHeaderPrefix + prefix));
    }

    public ValueProviderResult GetValue(string key)
    {
        IEnumerable<string> values;

        return _headers.TryGetValues(XHeaderPrefix + key, out values)
            ? new ValueProviderResult(values.First(), values.First(), CultureInfo.CurrentCulture)
            : null;
    }
}

public class XHeaderValueProviderFactory : ValueProviderFactory
{
    public override IValueProvider GetValueProvider(HttpActionContext actionContext)
    {
        return new XHeaderValueProvider(actionContext);
    }
}
```

All that remains is to specify that each of our parameters are using the `XHeaderValueProviderFactory` to bind the values of the parameters:

``` csharp
[HttpGet]
[Route("customers/headers")]
public HttpResponseMessage GetViaHeaders([ValueProvider(typeof(XHeaderValueProviderFactory))] int pageNo = 1, [ValueProvider(typeof(XHeaderValueProviderFactory))] int pageSize = 50)
{
	...
}
```

Now when we call call the web method, simply request a specific page number and page size by specifying the `X-PageNo` and `X-PageSize` headers in the request, and our response will return the result with the correct paging information:

![](/assets/images/2015-02-18-paging-in-aspnet-webapi-http-headers/postman-request-headers.png)

## Conclusion

In this blog post I built on the previous example by changing the way in which paging information is requested and returned. Instead of using query string parameters to request a specific page, we now do it via an HTTP header. We also removed the JSON envelope and instead return the paging information in the HTTP response's headers.
