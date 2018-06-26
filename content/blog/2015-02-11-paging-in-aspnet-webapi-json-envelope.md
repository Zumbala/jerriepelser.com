---
date: 2015-02-11T00:00:00Z
description: |
  Looking at how we can implement paging in ASP.NET Web API by wrapping the result set in a simple JSON envelope containing the data as well as paging meta data.
tags:
- aspnet
- aspnet mvc
- aspnet web api
title: "Paging in ASP.NET Web API: Using a JSON Envelope"
url: /blog/paging-in-aspnet-webapi-json-envelope/
---

## Introduction
In the introductory post I looked at some of the various methods which some of the popular web applications use in their APIs to implement paging. In this post I will implement one of the more basic methods, namely how to return the list of records inside an envelope which contain paging information. 

The end user of the API will make a request such as the following:

```text
GET customers?offset=50&limit=50
```

So in other words, the response we are looking for is something like the following: 

```json
{
   "data":
	[
	   {
	       "id": 1,
			...
	   },
	   {
	       "id": 25,
			...
	   },
	   {
	       "id": 24,
			...
	   },
		...
	],
   "pagination":
   {
       "total": 220,
       "limit": 50,
       "offset": 50,
       "returned": 50
   }
}
```

## Paging using an offset and limit

In the sample code which accompanies this project, I have a web project set up with a `Customer` class. I also have a `Seed()` method which runs with the database migrations which will pre-populate my database with 220 fictitious customers. If you want to play with the project on your own computer, be sure to run the database migrations from the Package Manager Console:

``` text
PM> Update-Database
```

First thing to do is to create a `CustomersController` class which will contain the API methods. In the constructor of the class I create an instance of the database context which will be used:

```csharp
public class CustomersController : ApiController
{
    private readonly ApplicationDbContext _dbContext;

    public CustomersController()
    {
        _dbContext = new ApplicationDbContext();    
    }
}
```

The method to get customers will accept parameters called `offset` and `limit` which will allow the user to page through customer records:

``` csharp
public IHttpActionResult Get(int offset = 0, int limit = 50)
{
	...
}
```
To select the requested records based on the parameters is as easy as pie, as LINQ already contains methods called `Skip` and `Take` which allows your skip a certain number of elements and limit the number of elements returned. Read more on this in [the official documentation](https://msdn.microsoft.com/en-us/library/bb386988.aspx). So applying these two methods, this is what our method looks like:

``` csharp
public IHttpActionResult Get(int offset = 0, int limit = 50)
{
    // Select the customers based on paging parameters
    var customers = _dbContext.Customers
        .OrderBy(c => c.Id)
        .Skip(offset)
        .Take(limit)
        .ToList();

    // Return the list of customers
    return Ok(customers);
}
```

At this point we can page through the records, but the method only return an array of customers without any sort of envelope which contains the paging information. The controller action will need to retrieve the total number of records and then return the data according to the structure in the example listed above. I will keep it simple for now and just return an anonymous type which the data:

``` csharp
public IHttpActionResult Get(int offset = 0, int limit = 50)
{
    // Get total number of records
    int total = _dbContext.Customers.Count();

    // Select the customers based on paging parameters
    var customers = _dbContext.Customers
        .OrderBy(c => c.Id)
        .Skip(offset)
        .Take(limit)
        .ToList();

    // Return the list of customers
    return Ok(new
    {
        Data = customers,
        Paging = new
        {
            Total = total,
            Limit = limit,
            Offset = offset,
            Returned = customers.Count
        }
    });
}
``` 

And heading over to [Postman](http://www.getpostman.com/) and making the call to my Web API, you can see an example of the data returned in the screenshot below:

![](/assets/images/2015-02-11-paging-in-aspnet-webapi-json-envelope/postman-paging-by-offset.png)

## Paging using a page number and page size

I also want to show how easy it is to adjust the paging from an offset based paging mechanism to a page based mechanism. In the example above I specified `offset` and `limit` parameters to page through the data. In this section I will be demonstrating how you can specify `pageNo` and `pageSize` parameters instead to page through the records, e.g.:

```text
GET customers?pageno=5&pagesize=50
```

The response we are looking for will look like this: 

```json
{
   "data":
	[
	   {
	       "id": 1,
			...
	   },
	   {
	       "id": 25,
			...
	   },
	   {
	       "id": 24,
			...
	   },
		...
	],
   "paging":
   {
       "pageNo": 5,
       "pageSize": 50,
       "pageCount": 5
       "totalRecordCount": 220,
   }
}
```

This time instead of returning the data wrapped in an anonymous type, I will create a `PagedResult` class to encapsulate the records as well as the paging information:

``` csharp
public class PagedResult<T>
{
    public class PagingInfo
    {
        public int PageNo { get; set; }

        public int PageSize { get; set; }

        public int PageCount { get; set; }

        public long TotalRecordCount { get; set; }

    }
    public List<T> Data { get; private set; }

    public PagingInfo Paging { get; private set; }

    public PagedResult(IEnumerable<T> items, int pageNo, int pageSize, long totalRecordCount)
    {
        Data = new List<T>(items);
        Paging = new PagingInfo
        {
            PageNo = pageNo,
            PageSize = pageSize,
            TotalRecordCount = totalRecordCount,
            PageCount = totalRecordCount > 0
                ? (int) Math.Ceiling(totalRecordCount/(double) pageSize)
                : 0
        };
    }
}
```

I will also make a slight change to my controller action to firstly determine the number of records to skip (i.e. the offset) based on the page number and page size, and then also to return the records in an instance of the `PagedResult` class:

``` csharp
public IHttpActionResult GetPaged(int pageNo = 1, int pageSize = 50)
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
    return Ok(new PagedResult<Customer>(customers, pageNo, pageSize, total));
}
```

Heading over to Postman again to call the Web API, you can see an example of the data returned in the screenshot below:

![](/assets/images/2015-02-11-paging-in-aspnet-webapi-json-envelope/postman-paging-by-page.png)

## Conclusion

I demonstrated paging used both offset based or paged based methods. So which is better? The answer is that it depends on your needs. I personally prefer using paged based paging because it seems to me like a simpler method to use and easier for the end user to understand. However, if your end users need to be able to select records from any random offset (for whatever obscure reason), then a paged based mechanism is not going to work for you. In such an instance it would be better to implement offset based paging as it allows for much more flexibility.

Either way is quite easy to implement and whichever way you choose comes down to your personal preference and the specific requirements of your API.

Next time we will look at sending back paging information in the HTTP headers, instead of wrapping your result in an envelope.
