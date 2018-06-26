---
date: 2014-10-28T00:00:00Z
description: |
  Demonstrates how to build a grid with sorting and paging using ASP.NET Web API, AngularJS, Restangular and ng-Table. This is Part 4 which adds sorting.
tags:
- aspnet
- aspnet web api
- angularjs
- ngTable
title: 'Building an interactive grid with ASP.NET Web API and AngularJS: Add sorting
  (Part 4)'
url: /blog/webapi-angular-grid-part-4/
---

In [Part 3](http://www.jerriepelser.com/blog/webapi-angular-grid-part-3) of the series we finally added ngTable into the mix which allows us to display the data in a table and also page through the data. In this blog post we will look at adding sorting capabilities to our table.

## Setting ngTable up for sorting
Allowing users to sort by a column in ngTable is as easy as adding a `data-sortable` attribute to each column for which we want to allow sorting. The `data-sortable` also need to specify the name of the field by which will be sorted. So open the `Index.cshtml` file and change the two existing columns to add sorting:

```html
<table ng-table="vm.tableParams" class="table">
    <tr ng-repeat="customer in $data">
        <td data-title="'First Name'" data-sortable="'firstName'">{{customer.firstName}}</td>
        <td data-title="'Last Name'" data-sortable="'lastName'">{{customer.lastName}}</td>
    </tr>
</table>
```

When you now run the application you will notice that the columns has sort indicators.

![](/assets/images/webapi-angular-grid/sorting.png)

Clicking on a column will toggle the sort indicator.

![](/assets/images/webapi-angular-grid/sorting-2.png)

## Passing the sorting parameters to the API
Now that a user can click on a column to sort by that column, we need to pass the actual sort order through to the API. To do that we go back to the `CustomersController` and where we pass the parameters for the page number and page size, we also pass the current sorting order as supplied by ngTable by accessing the `orderBy()` method of the `params` argument.

```javascript
vm.tableParams = new ngTableParams({
    ...
},
{
	getData: function ($defer, params) {
	    // Load the data from the API
	    Restangular.all('customers').getList({
	        pageNo: params.page(),
	        pageSize: params.count(),
	        sort: params.orderBy()
	    }).then(function (customers) {
	        ...
	    }, function (response) {
	        ...
	    });
	}
});

```

We can also specify the default sorting order when the page loads by passing it in the `parameters` argument of the `ngTableParams` function.

```javascript
vm.tableParams = new ngTableParams({
    page: 1,
    count: 10,
    sorting: {
        lastName: 'asc'
    }
},
{
    getData: function ($defer, params) {
        ...
    }
});
```

If you now look at the Network calls being made back to the API when you click on a column, you will notice that a `sort` parameter is passed along which specifies the column to be sorted by. The sort direction is also added by prefixing a `+` or `-` before the column name. So if we sort by Last Name ascending, the following request will be made to the API:

```text
/api/customers?pageNo=1&pageSize=10&sort=-lastName
```

If you sort by multiple columns (by holding the Ctrl key when clicking on a column) you will see that multiple sort parameters gets passed along.

```text
/api/customers?pageNo=1&pageSize=10&sort=-lastName&sort=-firstName
```

## Adding sorting to the API
The final step is to change our API method to also take a string array paramater called `sort` which will contain the sort parameters being passed from the grid. As the parameters are passed in the query string it is important to specify that `sort` parameter needs to be bound from the URI. To do this we add the `FromUri` attribute to the `sort` parameter. 

```csharp
public PagedResult<Customer> Get(int pageNo = 1, int pageSize = 50, [FromUri] string[] sort = null)
{
	...
}
```

The field names for the sorting is being passed in as strings, so to make my life a little bit easier I am making use of the [System.Ling.Dynamic](https://github.com/kahanu/System.Linq.Dynamic) Nuget package to assist in adding "dynamic" sorting to a LINQ query at runtime by passing the sorting definition as a string, and not as a Lambda expression. 

So instead of specifying sorting like this:

```csharp
var customers = demoContext.Customers.OrderBy(c => c.LastName);
```

I can specify it like this:

```csharp
var customers = demoContext.Customers.OrderBy("LastName DESC");
```

Or even as a combination of fields like this:
 
```csharp
var customers = demoContext.Customers.OrderBy("LastName DESC, FirstName DESC");
```

So go ahead and install the package:


```text
Install-Package System.Linq.Dynamic
```

The next step is to take the string array with the sorting columns and convert it to a string sorting expression, so an array like this:

```text
sort[0] = "-LastName";
sort[1] = "-FirstName";

```

Will be turned into a string like this:

```csharp
"LastName DESC, FirstName DESC"
```

For that I write a little extension method.

```csharp
public static class QuerySortingExtensions
{
    public static IQueryable<T> ApplySorting<T>(this IQueryable<T> query, IEnumerable<string> sort) where T : class
    {
        if (sort != null)
        {
            List<string> sortFields = new List<string>();

            foreach (string sortField in sort)
            {
                string direction = null;
                string fieldName = null;

                if (sortField.StartsWith("+"))
                {
                    sortFields.Add(string.Format("{0} ASC", sortField.TrimStart('+')));
                }
                else if (sortField.StartsWith("-"))
                {
                    sortFields.Add(string.Format("{0} DESC", sortField.TrimStart('-')));
                }
                else
                {
                    sortFields.Add(sortField);
                }
            }

            return query.OrderBy(String.Join(",", sortFields));
        }

        return query;
    }
}
```

And change the querying code in my controller to call this extension method to apply the sorting order passed in via the `sort` parameter:

```csharp
public PagedResult<Customer> Get(int pageNo = 1, int pageSize = 50, [FromUri] string[] sort = null)
{
    // Determine the number of records to skip
    int skip = (pageNo - 1) * pageSize;

    IQueryable<Customer> queryable = demoContext.Customers;

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

You will notice that I added a check to see whether the sort parameter was specified, and if so I call the extension method to convert the sort order array into a string which is understood by the Dynamic LINQ library and apply that to the query. When using the `Skip()` LINQ method of the Entity Framework you always need to specify a sort order, so if no sort order was specied I default to sort by the `Id` column. 

And that all there is to it. It looks a little complex because there are so many pieces coming together, but it really is fairly simple. If you are looking at this and going "Dude, WTF?" then please grab the source code from GitHub, put a few breakpoints where something does not make sense to you to debug and gain a better understanding.

In [Part 5](http://www.jerriepelser.com/blog/webapi-angular-grid-part-5) we will be adding searching.

