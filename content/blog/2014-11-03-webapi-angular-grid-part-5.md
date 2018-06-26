---
date: 2014-11-03T00:00:00Z
description: |
  Demonstrates how to build a grid with sorting and paging using ASP.NET Web API, AngularJS, Restangular and ng-Table. This is Part 5 which adds searching.
tags:
- aspnet
- aspnet web api
- angularjs
- ngTable
title: 'Building an interactive grid with ASP.NET Web API and AngularJS: Add searching
  (Part 5)'
url: /blog/webapi-angular-grid-part-5/
---

In [Part 4](http://www.jerriepelser.com/blog/webapi-angular-grid-part-4) of this series, I added sorting to the table. In this blog post I will add a search box which allows users to search for customers by first name or last name. Whenever a user types something into the search box we will automatically trigger a new request to the API to filter the customers based on the text the user entered. 

## Add a search box

The first step is very simple and that is to add a text box to the existing page.

```html
<div ng-controller="CustomersController as vm">
    <h1>Customers</h1>
    <p>
        <input class="form-control" placeholder="Search by name" type="text" ng-model="vm.search" ng-model-options="{ debounce: 500 }" />
    </p>
    <table ng-table="vm.tableParams" class="table">
        ...
    </table>
</div>
```

The text box is bound to a `search` field on the Angular controller and I also make use of the [ngModelOptions](https://docs.angularjs.org/api/ng/directive/ngModelOptions) directive in AngularJS 1.3 to specify a `debounce` 
value of 500. This means that when a user start typing in the input box, the underlying model will only get updated once the user has stopped typing for 500ms, so we don't make a call to the API every time the user presses a key.

## Updating the Angular Controller
Next up we need to update the `CustomersController` for the Angular application to pass the search parameter through to the API when Restangular makes the call to fetch customers:

```javascript
...
getData: function ($defer, params) {
    // Load the data from the API
    Restangular.all('customers').getList({
        pageNo: params.page(),
        pageSize: params.count(),
        sort: params.orderBy(),
        search: vm.search
    }).then(function (customers) {
        ...
    }, function (response) {
        ...
    });
}
...
```

Note that I have added a `search` parameter which is bound to the `search` field from the controller. This will however not automatically trigger `ngTable` to reload the data in the table. For that we need to add a watch to the scope, and up until now we did not make use of a scope as we used the **Controller As** syntax. So simply inject `$scope` into the controller and add a `$watch` on the `search` field.

```javascript
$scope.$watch(angular.bind(vm, function () {
    return vm.search;
}), function (value) {
    vm.tableParams.reload();
});
```  

The final full source code for the controller looks like this:

```javascript
(function () {
    'use strict';

    angular
        .module('app')
        .controller('CustomersController', CustomersController);

    CustomersController.$inject = ['$scope', 'Restangular', 'ngTableParams'];

    function CustomersController($scope, Restangular, ngTableParams) {
        /* jshint validthis:true */
        var vm = this;

        vm.search = '';

        vm.tableParams = new ngTableParams({
            page: 1,
            count: 10,
            sorting: {
                lastName: 'asc'
            }
        },
        {
            getData: function ($defer, params) {
                // Load the data from the API
                Restangular.all('customers').getList({
                    pageNo: params.page(),
                    pageSize: params.count(),
                    sort: params.orderBy(),
                    search: vm.search
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

        // Watch for changes to the search text, so we can reload the table
        $scope.$watch(angular.bind(vm, function () {
            return vm.search;
        }), function (value) {
            vm.tableParams.reload();
        });
    }
})();
```

## Update the Web API method
The last bit is to add the web api to accept the search parameter and filter customers based on that. Add a `search` the parameter to the existing method and update the method to filter the customers based on the search string:

```csharp
public PagedResult<Customer> Get(int pageNo = 1, int pageSize = 50, [FromUri] string[] sort = null, string search = null)
{
    // Determine the number of records to skip
    int skip = (pageNo - 1) * pageSize;

    IQueryable<Customer> queryable = demoContext.Customers;

    // Apply the search
    if (!String.IsNullOrEmpty(search))
    {
        string[] searchElements = search.Split(new char[] { ' ' }, StringSplitOptions.RemoveEmptyEntries);
        foreach (string searchElement in searchElements)
        {
            string element = searchElement;
            queryable = queryable.Where(c => c.FirstName.Contains(element) || c.LastName.Contains(element));
        }
    }

    // Add the sorting
    if (sort != null)
        queryable = queryable.ApplySorting(sort);
    else
        queryable = queryable.OrderBy(c => c.Id);

    // Get the total number of records
    int totalItemCount = queryable.Count();

    // Retrieve the customers for the specified page
    var customers = queryable
        .Skip(skip)
        .Take(pageSize)
        .ToList();

    // Return the paged results
    return new PagedResult<Customer>(customers, pageNo, pageSize, totalItemCount);
}
```

Note that I take whatever the search string is and split it into multiple words whever a space character appears. I then take each of those search words and check to see whether either the first name or	 last name contains that word and filter out customers accordingly. This allows users to search on a combination of first name and last name.

> **Please note** that because my search uses the `IndexOf` function it effectively translates into doing the SQL search `WHERE FirstName LIKE '%search term%'` which is not efficient as it will not use any indexes. If you implement something like this into a production system you should definitely use more effective mechanism such as SQL Server's full text searching or some other proper search engine

With that in place you can run the application and play with the new search capabilities:

![](/assets/images/webapi-angular-grid/customer-list-part-5.png)

