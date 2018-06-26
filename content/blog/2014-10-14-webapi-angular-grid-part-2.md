---
date: 2014-10-14T00:00:00Z
description: |
  Demonstrates how to build a grid with sorting and paging using ASP.NET Web API, AngularJS, Restangular and ng-Table. This is Part 2 which adds a very basic grid.
tags:
- aspnet
- aspnet web api
- angularjs
title: 'Building an interactive grid with ASP.NET Web API and AngularJS: A basic grid
  (Part 2)'
url: /blog/webapi-angular-grid-part-2/
---

## Introduction
In [Part 1](http://www.jerriepelser.com/blog/webapi-angular-grid-part-1) I covered the introduction to this series and also went through the basic setup to add the required Javascript libraries and added the basis for our AngularJS application. In this blog post I will go through adding the Entity Framework model, the API for retrieving a list of customer and also hooking up the AngularJS frontend to the Web API and display the list of customers on the page.

## Creating the Entity Framework models
The data model is going to be very basic and include only a `Customer` entity with a first name and last name. I am not going to go though the process of creating the Entity Framework model in much detail, so if you are not familiar with this please have a look at Tom Dykstra's tutorial on the ASP.NET website entitled [Getting Started with Entity Framework 6 Code First using MVC 5](http://www.asp.net/mvc/tutorials/getting-started-with-ef-using-mvc/creating-an-entity-framework-data-model-for-an-asp-net-mvc-application).

Make sure that you have the Entity Framework Nuget package installed so from the **Package Manager Console**, execute:

```bash
Install-Package EntityFramework
```

Create a Customer class:

```csharp
public class Customer
{
    [Key]
    public Guid Id { get; set; }

    [Required]
    public string FirstName { get; set; }

    [Required]
    public string LastName { get; set; }
}
```

Create the database context:

```csharp
public class DemoContext : DbContext
{
    public DbSet<Customer> Customers { get; set; }

    public DemoContext()
        : base("DemoContext")
    {
    }
}
```

And finally register the connection string in the `web.config` file:

```xml
<connectionStrings>
	<add name="DemoContext"
     	connectionString="Data Source=(LocalDb)\v11.0;Initial Catalog=DemoDatabase;Integrated Security=SSPI;"
     	providerName="System.Data.SqlClient" />
</connectionStrings>
``` 

### Creating some sample data
We are going to need some demo data for this application. For this I am going to create a database initializer and use 2 other Nuget packages to assist me in creating the demo data, namely [NBuilder](https://code.google.com/p/nbuilder/) and [Faker.NET](https://github.com/oriches/faker-cs). NBuilder helps to rapidly create test objects, and Faker.NET helps to generate meaningful values for the fields of the test objects, in our case proper first names and last names.

Open the **Package Manager Console** again and install the 2 packages: 

```bash
Install-Package nbuilder
```

```bash
Install-Package faker.net
```

Create a database initializer for the context which is going to initialize the database and create a `Seed()` which uses NBuilder to create a list of 200 customers and add them to the database.

```csharp
public class DemoInitializer : DropCreateDatabaseIfModelChanges<DemoContext>
{
    protected override void Seed(DemoContext context)
    {
        var customers = Builder<Customer>.CreateListOfSize(200)
            .All()
                .With(c => c.FirstName = Faker.Name.First())
                .With(c => c.LastName = Faker.Name.Last())
            .Build();

	    foreach (var customer in customers)
	    {
	        context.Customers.AddOrUpdate(c => c.Id,
	            customer);
	    }
    }
}
```

What the seed method does is use NBuilder to create a list of customers. Normally NBuilder will fill string fields with random characters, but I want to have some semi-decent looking data, so I use Faker to generate "proper" first and last names for the customers.

I then loop through the list of customers and use the data context to add them to the database.

If the stuff I do with NBuilder does not make sense to you, please look at the [NBuilder home page](https://code.google.com/p/nbuilder/) for some examples and an explanation.  

To tell Entity Framework to use the database initializer, I need to register it in the `web.config`.

```xml
<entityFramework>
  <contexts>
    <context type="AngularWebApiGrid.Models.DemoContext, AngularWebApiGrid">
      <databaseInitializer type="AngularWebApiGrid.Models.DemoInitializer, AngularWebApiGrid" />
    </context>
  </contexts>
  ...
</entityFramework>
```

Now, when I use the data context, Entity Framework will call the database initializer and drop and re-create the database if the model changes (not something you want to do in a production system!) and also run the seed method.  

## Creating the API method
For now I am going to keep the API method to retrieve the list of customers very simple and only return a list with all the customers. I will deal with the paging next time.

So create a new Web API controller called `CustomersController`. Create a single method called `Get` which returns all the customers from the database.

```csharp
public class CustomersController : ApiController
{
    private readonly DemoContext demoContext;

    public CustomersController()
    {
        demoContext = new DemoContext();
    }

    // GET api/<controller>
    public IEnumerable<Customer> Get()
    {
        return demoContext.Customers;
    }
}
```

Go the the URL for the API in your browser or in a tool like Postman and behold a beautiful list of customers:

![](/assets/images/webapi-angular-grid/postman-customer-list.png)

## Using Restangular to retrieve the data from the API
For now our backend is complete. We have and Entity Framework model for our list of customers, we initialize the database with some demo customers and have an API method to retrieve those customers.

Now we head back to the Angular application and alter the Customers controller to retrieve the list of customers when it initializes. You may remember from the previous time that the SideWaffle template created a nice basic structure for the controller with an activate method which get called automatically when the controller is instantiated.

At this point I suggest you go and have a [look at the documentation for Restangular](https://github.com/mgonto/restangular) to familiarise yourself a little bit more with what it is and how it works. I will give you a 10 second intro.

Basically Restangular is a library for Angular which makes working with RESTful APIs much easier. For example if I want to retrieve my list of customers, all I have to do is add the line of code:

```javascript
Restangular.all('customers').getList()
``` 

and Restangular will automatically make the call to `GET: /customers` to our backend API to retrieve the list of customers. It is important to note that Restangular use promises, so the proper code to retrieve our list of customers would be:

```javascript
Restangular.all('customers').getList().then(function(customers) {
	// Do something with the list of customers we retrieved...
});
```

So with that quick little intro, here is the code for the Customers controller:

```javascript
(function () {
    'use strict';

    angular
        .module('app')
        .controller('CustomersController', CustomersController);

    CustomersController.$inject = ['Restangular']; 

    function CustomersController(Restangular) {
        /* jshint validthis:true */
        var vm = this;
        vm.customers = [];

        activate();

        function activate() {
            Restangular.all('customers').getList().then(function(customers) {
                vm.customers = customers;
            });
        }
    }
})();
```

A few things to note:
* In order to use the Restangular object you need to inject it into the controller. We already specified the dependency of our module on Restangular in the previous blog post.
* I declare an empty array on my view model called `customers` which will hold the list of customers.
* In the `activate()` method I use Restangular to retrieve the list of customers and when the promise is resolved and the list is returned I assign the customers to the customers array on my view model.

At this point there is one final piece of Restangular configuration which is required. If I use the code like specifed above, Restangular will attempt to retrieve the list of customers on the URL `/customers`. But the correct URL for the customers is `/api/customers`. Thankfully Restangular allows us to specify a base URL for all calls. To do this open the `app.js` file and add a configuration block to the Angular module which calls the `RestangularProvider.setBaseUrl()` method to specify the base URL for all Restangular calls: 

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
    };


})();
```

All that is left now is to display the list of customers. Head over to the `\Views\Home\Index.cshtml` file and change the existing markup to simply loop through the list of customers and display the first name and last name for each customer.

```html
<div ng-controller="CustomersController as vm">
    <h1>Customers</h1>    
    <p ng-repeat="customer in vm.customers">
        {{ customer.FirstName + ' ' + customer.LastName }}
    </p>
</div>

```

With that you can run the project and you should now see an HTML page which the list of customers displayed:

![](/assets/images/webapi-angular-grid/customer-list-part-2.png)

## A few notes
I must just point a few things out quickly. This is not the sort of code you really want to write in a production system. For one thing you do not want to drop and recreate the database in the database initializer every time. You would also probably want to use dependency injection to inject the database context into your Web API controllers. Another thing is that you really should not return your Entity Framework models from the API methods. You would probably want to create some sort of [DTO](https://en.wikipedia.org/wiki/Data_transfer_object) and use a tool like AutoMapper to map between the DTO and the Entity Framework objects.

I keep things simple to demonstrate a specific technique and introducing things like dependency injection and AutoMapper is just going to distract from what I am really trying to demonstrate. You are a professional developer, so use your own better judgement in matters like these.

I am also skipping quickly over a lot of things otherwise these blog posts are going to become terribly long. I assume you are proficient enough with the internet to go read up in more detail where I may have touched on certain things too briefly for your liking. I also assume you are proficient enough with a debugger to step through things slowly to better understand what is happening at runtime.

## Conclusion
In this blog post I created an Entity Framework model for customers and a Web API which returns those list of customers. I also demonstrated how to use Restangular to communicate with the Web API to retrieve the list of customers and then used the Angular `ngRepeat` directive to display the list of customers.

In [Part 3](http://www.jerriepelser.com/blog/webapi-angular-grid-part-3) I will add paging to the customers API and also replace the simple list with ngTable.
