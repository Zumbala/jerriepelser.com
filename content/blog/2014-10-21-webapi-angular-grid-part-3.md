---
date: 2014-10-21T00:00:00Z
description: |
  Demonstrates how to build a grid with sorting and paging using ASP.NET Web API, AngularJS, Restangular and ng-Table. This is Part 3 which adds paging and ngTable.
tags:
- aspnet
- aspnet web api
- angularjs
- ngTable
title: 'Building an interactive grid with ASP.NET Web API and AngularJS: Add paging
  and ngTable (Part 3)'
url: /blog/webapi-angular-grid-part-3/
---

In [Part 2](http://beabigrockstar.com/blog/webapi-angular-grid-part-2) I created an ASP.NET Web API with a single method that returns a list of customers. I then went on to show how to use Restangular to retrieve the list of customers and display them. In this blog post I will change the API method which retrieves the list of customers to allow for paging.

## Fix JSON formatting

The first thing I want to do however is make a small change to the API. Currently the JSON being returned by the API is formatted as Pascal case, but the more general convention is for JSON documents to be formatted with Camel case. I like to stick to this convention and to make this change is quite easy.

To change to way the ASP.NET Web API formats JSON you need to edit the `WebApiConfig` class and specify the `ContractResolver` as an instance of the `CamelCasePropertyNamesContractResolver` class. This will force the JSON serializer to format all the properties of the `Customer` class as Camel case.    

```csharp
public static class WebApiConfig
{
    public static void Register(HttpConfiguration config)
    {
        // Configure JSON formatter
        var settings = config.Formatters.JsonFormatter.SerializerSettings;
        settings.ContractResolver = new CamelCasePropertyNamesContractResolver();

		...
    }
}
```

After the change a `Customer` returned from the API will look like this:

```json
{
	id: "00000000-0000-0000-0000-00000000004d",
	firstName: "Arlie",
	lastName: "Abshire"
}
```

And of course you need to take care to change the casing of the property names in the Angular template as well:

```html
<div ng-controller="CustomersController as vm">
    <h1>Customers</h1>    
    <p ng-repeat="customer in vm.customers">
        {{ customer.firstName + ' ' + customer.lastName }}
    </p>
</div>
```

We are actually going to replace the Angular template above with the `ngTable` implementation, so the last step is not really needed as that code is going to be deleted in any case. 

## Add paging to customer list API

Currently the API method which returns a list of customers simply return the customers as a `IEnumerable<Customer>`. I want to add paging to the API so you can request a specific page number and also specify the number of items per page to return. I also want to return a paged result set from the API call with some extra meta data regarding the page returned. The paged result set will look as follows:

```csharp
public class PagedResult<T>
{
    public int PageNo { get; set; }

    public int PageSize { get; set; }

    public int PageCount { get; private set; }

    public long TotalRecordCount { get; set; }

    public PagedResult(IEnumerable<T> items, int pageNo, int pageSize, long totalRecordCount)
    {
        Items = new List<T>(items);

        PageNo = pageNo;
        PageSize = pageSize;
        TotalRecordCount = totalRecordCount;

        PageCount = totalRecordCount > 0
                    ? (int)Math.Ceiling(totalRecordCount / (double)PageSize)
                    : 0;
    }

    public List<T> Items { get; set; }
}
```

The result set will indicate the number of the page which was returned, the size of a page (i.e. the number of records contained in each page), the total number of pages as well as the total number of records. Lastly it will also contain a list of the actual items.

I also change the API method to take two extra parameters so you can specify the page number you want as well as the number of records per page. The rest of the method is just a little bit of simple math to calculate the number of records to skip as well as changing the actual query to retrieve the specifed page.  

```csharp
public PagedResult<Customer> Get(int pageNo = 1, int pageSize = 50)
{
    // Determine the number of records to skip
    int skip = (pageNo - 1) * pageSize;

    // Get the total number of records
    int totalItemCount = demoContext.Customers.Count();

    // Retrieve the customers for the specified page
    var customers = demoContext.Customers
        .OrderBy(c => c.LastName)
        .Skip(skip)
        .Take(pageSize)
        .ToList();

    // Return the paged results
    return new PagedResult<Customer>(customers, pageNo, pageSize, totalItemCount);
}
```

After that you can head back to Postman to look at the new JSON returned from the API call:

![](/assets/images/webapi-angular-grid/postman-paged-data.png)

The default page size is 50 records. To select a specific page, just specify the parameters for the page number and page size:

![](/assets/images/webapi-angular-grid/postman-paged-data-2.png)

## Extract paged data in Restangular

If you try and run the application now you will notice that you get a Javascript error:

![](/assets/images/webapi-angular-grid/restangular-getlist-expects-array.png)

The reason for this is that we request a list of objects through Restangular (by making use of the `getList()` function), and Restangular therefore expects an array on items to be retrieved. We are however not returning an array of items from the API method anymore but out custom object containing the paged results with the actual list of customers contained in the `items` property of that JSON object. Thankfully Restangular has a solution for this as described in the FAQ: https://github.com/mgonto/restangular#my-response-is-actually-wrapped-with-some-metadata-how-do-i-get-the-data-in-that-case

So what we need to do is add a response interceptor in the Restangular configuration to check whether we are retrieving a list of items, as these are the ones which will be wrapped is some paging meta data. If it is a list we extract the array of items from the `items` property of the data returned from the API call and return that. We also extract the actual paging meta data and add that as an extra `paging` property which can be accessed later on and passed on to `ngTable` to enable it to display the paging information correctly. 

```javascript
(function () {
    'use strict';

    angular.module('app', [
        // Angular modules 

        // Custom modules 

        // 3rd Party Modules
        'restangular',
        'ngTable'
    ])
    .config(restangularConfig);

    restangularConfig.$inject = ['RestangularProvider'];

    function restangularConfig(RestangularProvider) {
        RestangularProvider.setBaseUrl('/api');

        RestangularProvider.addResponseInterceptor(function (data, operation, what, url, response, deferred) {
            var extractedData;
            // .. to look for getList operations
            if (operation === "getList") {
                // .. and handle the data and meta data
                extractedData = data.items;
                extractedData.paging =
                {
                    pageCount: data.pageCount,
                    pageNo: data.pageNo,
                    pageSize: data.pageSize,
                    totalRecordCount: data.totalRecordCount
                };
            } else {
                extractedData = data;
            }
            return extractedData;
        });
    };
})();
```

## Add ngTable

The last piece of the puzzle is to add [ngTable](http://bazalt-cms.com/ng-table/). 

I am no going to go through a detailed step-by-step explanation on how to configure ngTable at its basic level. I suggest you read the document entitled [Configuring your table with ngTableParams](https://github.com/esvit/ng-table/wiki/Configuring-your-table-with-ngTableParams) on their Wiki to see how to configure a `ngTableParams` object in your controller and link that to your Angular view.

I also suggest you look throug the examples on the website to see how to use it, and pay particular attention to [Example #6](http://bazalt-cms.com/ng-table/example/6), as that demonstrates how to retrieve data via AJAX and we will use that same method to call Restangular.

So assuming you now have a very basic understanding of how use ngTable let us change the existing code in our controller and view to use ngTable.

Here is what our final controller code look like. I will discuss in more detail below.

```javascript
(function () {
    'use strict';

    angular
        .module('app')
        .controller('CustomersController', CustomersController);

    CustomersController.$inject = ['Restangular', 'ngTableParams'];

    function CustomersController(Restangular, ngTableParams) {
        /* jshint validthis:true */
        var vm = this;

        vm.tableParams = new ngTableParams({
            page: 1,
            count: 10
        },
        {
            getData: function ($defer, params) {
                // Load the data from the API
                Restangular.all('customers').getList({
                    pageNo: params.page(),
                    pageSize: params.count()
                }).then(function (customers) {
                    // Tell ngTable how many records we have (so it can set up paging)
                    params.total(customers.paging.totalRecordCount);

                    // Return the customers to ngTable
                    $defer.resolve(customers);
                }, function (response) {
                    // Notify of error
                });
            }
        });

        activate();

        function activate() {
        }
    }
})();
```   

If your read the documentation on the ngTable Wiki you would have seen the the `ngTableParams` function takes two arguments. The first is a parameters object which specify certain default values, and in our case I specified the default page number to request as page 1, and also a default page size of 10 records.

The second argument is a settings object and for the settings object and if you looked at the AJAX Example on the ngTable website you would have noticed that they specify a `getData` function on the settings object to retrieve the data via an AJAX call. In our case we specify a `getData` function which uses Restangular to retrieve the list of customers.

The Restangular call also changes a bit to now pass the page number and page size parameters to the API method. You will recall from the previous blog post that to retrieve the list of customers we would make a call like this: 

```javascript
Restangular.all('customers').getList().then(function(customers) {
	// Do something with the list of customers we retrieved...
});
```

This makes the following call to the backend API:

```bash
GET: /api/customers
```

For the `getList()` method however you can specify an object with extra parameters to pass along to the API call, for example:   

```javascript
Restangular.all('customers').getList({ pageNo: 1, pageSize: 10 }).then(function(customers) {
	// Do something with the list of customers we retrieved...
});
``` 

Would pass the query string parameters `pageNo` and `pageSize` to the API method:

```bash
GET: /api/customers?pageNo=1&pageSize=10
```

So in our modified controller we use this method to request the specified page number and page size as requested by ngTable. ngTable will pass those in via the `params` argument of the `getData()` function:

```javascript
vm.tableParams = new ngTableParams({
    pageNo: params.page(),
    pageSize: params.count()
},
{
    getData: function ($defer, params) {
        Restangular.all('customers').getList({
            pageNo: params.page(),
            pageSize: params.count()
        }).then(function (customers) {
			...
        }, function (response) {
            ...
        });
    }
});
```

The `$defer` argument is a Promise object and we use the `resolve` method of this Promise object to return the list of objects to display in the grid to ngTable. The other thing we want to do is tell ngTable how many records there are in total so it can display the correct paging information at the bottom of the table. We use the `total()` method of the `params` argument to specify the total number of records. To retrieve the total number of records we simply get that from the `paging` meta data we added to the Restangular result before. 

```javascript
vm.tableParams = new ngTableParams({
    ...
},
{
    getData: function ($defer, params) {
        Restangular.all('customers').getList({
            pageNo: params.page(),
            pageSize: params.count()
        }).then(function (customers) {
		    // Tell ngTable how many records we have (so it can set up paging)
		    params.total(customers.paging.totalRecordCount);
		
		    // Return the customers to ngTable
		    $defer.resolve(customers);
        }, function (response) {
            ...
        });
    }
});
```

Finally change to Restangular view to display the list of customers using the `ng-table` directive:

```html
<table ng-table="vm.tableParams" class="table">
    <tr ng-repeat="customer in $data">
        <td data-title="'First Name'">{{customer.firstName}}</td>
        <td data-title="'Last Name'">{{customer.lastName}}</td>
    </tr>
</table>
```

That is all that is needed. You can now run the project again an you should see how ngTable has created a nice data grid with paging build in:

![](/assets/images/webapi-angular-grid/ng-table-grid.png)

In [Part 4](http://www.jerriepelser.com/blog/webapi-angular-grid-part-4) we will look at adding sorting and filtering.
