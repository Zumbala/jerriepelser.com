---
date: 2015-11-24T00:00:00Z
description: |
  I look at how you can use the new In-Memory database support in Entity Framwork 7 to unit test your controllers.
tags:
- asp.net
- asp.net 5
- entity framework
- unit testing
title: Unit testing ASP.NET 5 Controllers with Entity Framework 7 In-Memory Database
url: /blog/unit-testing-aspnet5-entityframework7-inmemory-database/
---

## Introduction

A number of months ago I wrote about how you could [unit test with Entity Framework using a fake DbSet and NBuilder](http://www.jerriepelser.com/blog/unit-testing-with-fake-dbset-nbuilder). It's been almost 4 months since that post and I have done quite a lot of work using ASP.NET 5 in the meantime. So I thought I would write an updated version of that blog post, using ASP.NET 5 and Entity Framework 7.

In this blog post I am going to show you a very simple scenario where I have a `ProductsController` class which displays a list of products, and also allows the user to select a product and navigate to a details page.

We'll then look at how we can unit test the controller to make sure the correct results are returned. Since the controller uses the database context directly I will demonstrate how you can use the new In-Memory Database option in Entity Framework 7, instead of having to mock the database context. I will also use [GenFu](http://genfu.io/) to generate some data to test against.

## The demo application

For the base application I have a simple product listing which displays a list of products in the database. The list has a basic filter which allows the user to include or exclude discontinued products from the list:

![](/assets/images/unit-testing-aspnet5-entityframework7-inmemory-database/product-list.png)

When a user clicks on the name of a product, they are navigated to the detail screen where they can view the details of a particular product:

![](/assets/images/unit-testing-aspnet5-entityframework7-inmemory-database/product-detail.png)

This is what the products controller looks like:

``` csharp
public class ProductsController : Controller
{
    private readonly ApplicationDbContext dbContext;

    public ProductsController(ApplicationDbContext dbContext)
    {
        if (dbContext == null)
            throw new ArgumentNullException(nameof(dbContext));

        this.dbContext = dbContext;
    }

    public ActionResult Index(bool includeDiscontinued = false)
    {
        // Set base product query
        var productsQuery = dbContext.Products.AsQueryable();

        // Filter out discontinued products
        if (!includeDiscontinued)
            productsQuery = productsQuery.Where(p => p.IsDiscontinued == false);

        // Create view model
        var viewModel = new ProductIndexViewModel
        {
            IncludeDiscontinued = includeDiscontinued,
            Products = productsQuery.ToList()
        };

        // Display index view
        return View(viewModel);
    }

    public ActionResult Details(int? id)
    {
        // Ensure an Id is passed in
        if (id == null)
            return new BadRequestResult();

        // Get the product from the database
        Product product = dbContext.Products.FirstOrDefault(p => p.Id == id);

        // Ensure the product exists
        if (product == null)
            return HttpNotFound();

        // Display the product details view
        return View(product);
    }
}
```

For the Index action I get the list of products, and filter out the discontinued ones in case the user does not want to view discontinued products (which is the default).

For the Details action I display the product with some checks to see whether an `id` was passed it, and also checking to see whether the product exists and returning appropriate HTTP status codes if either case is not true.

For completeness sake, here is my current `ApplicationDbContext` and `Product` class:

``` csharp
public class ApplicationDbContext : DbContext
{
    public DbSet<Product> Products { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Make Blog.Url required
        modelBuilder.Entity<Product>(builder =>
        {
            builder.Property(b => b.Name).IsRequired();
            builder.Property(b => b.Description).IsRequired();
        });
    }
}

public class Product
{
    public string Description { get; set; }
    public int Id { get; set; }
    public string ImageUrl { get; set; }
    public bool IsDiscontinued { get; set; }
    public string Name { get; set; }
    public float Price { get; set; }
}
```  

And also the `ProductIndexViewModel` which gets passed to the Index view:

``` csharp
public class ProductIndexViewModel
{
    public bool IncludeDiscontinued { get; set; }
    public List<Product> Products { get; set; }
}
```

## Creating the tests

For the unit tests I used XUnit, and I basically followed their [Getting Started with xUnit.net (DNX / ASP.NET 5)](https://xunit.github.io/docs/getting-started-dnx.html) guide. 

After creating the unit test project I added a reference to the main web project and also added the following Nuget packages:

``` bash
install-package GenFu
install-package FluentAssertions
```

This is the project.json for my test project:

``` json
{
  "version": "1.0.0-*",
  "description": "Aspnet5DbContextTesting.Tests Class Library",
  "authors": [ "jerriep" ],
  "tags": [ "" ],
  "projectUrl": "",
  "licenseUrl": "",
  "dependencies": {
    "Aspnet5DbContextTesting": "1.0.0-*",
    "EntityFramework.InMemory": "7.0.0-rc1-final",
    "GenFu": "1.0.4",
    "FluentAssertions": "4.0.1",
    "xunit": "2.1.0",
    "xunit.runner.dnx": "2.1.0-rc1-build204"
  },
  "commands": {
    "test": "xunit.runner.dnx"
  },
  "frameworks": {
    "dnx451": { },
    "dnxcore50": { }
  }
}
```

I added a `ProductsControllerTests` class and set up some basic dependency injection bootstrapping of the database context:

``` csharp
public class ProductsControllerTests
{
    private readonly IServiceProvider serviceProvider;

    public ProductsControllerTests()
    {
        var services = new ServiceCollection();

        services.AddEntityFramework()
            .AddInMemoryDatabase()
            .AddDbContext<ApplicationDbContext>(options => options.UseInMemoryDatabase());

        serviceProvider = services.BuildServiceProvider();
    }
}
```

This creates a `ServiceCollection` class to which I register all the appropriate Entiry Framework classes, and then I create a service provider which I can use inside my unit tests to retrieve an instance of the database context.

I also need a method to intialise some test data:

``` csharp
private void CreateTestData(ApplicationDbContext dbContext)
{
    var i = 0;
    var id = 1;
    GenFu.GenFu.Configure<Product>()
        .Fill(p => p.Id, () => id++)
        .Fill(p => p.IsDiscontinued, () =>
        {
            i++;

            if (i % 3 == 0)
                return true;

            return false;
        });

    var products = GenFu.GenFu.ListOf<Product>(20);

    dbContext.Products.AddRange(products);
    dbContext.SaveChanges();
}
```

This test method uses GenFu to create a list of 20 products which I add to the database context, so I have a pre-defined state against which I can write my tests. For the `IsDiscontinued` property I will make every third product discontinued.

With that in place this is what a typical unit test will look like:

``` csharp
// Arrange
var dbContext = serviceProvider.GetRequiredService<ApplicationDbContext>();
CreateTestData(dbContext);
var controller = new ProductsController(dbContext);

// Act

// Assert
```

So I get an instance of `ApplicationDbContext` from the service provider, then add the test data to the database context and finally instantiate an instance of the controller, passing the database context as parameter.

## Testing the Index action

For the index action, I want to test that toggling the `includeDiscontinued` parameter results in the correct data being returned. 

For the first unit test I pass `true` as the parameter, so in this case I expect all 20 products to be returned.

``` csharp
[Fact]
public void IndexShouldIncludeDiscontinued()
{
    // Arrange
    var dbContext = serviceProvider.GetRequiredService<ApplicationDbContext>();
    CreateTestData(dbContext);
    var controller = new ProductsController(dbContext);

    // Act
    var actionResult = controller.Index(true);

    // Assert
    actionResult.Should().BeOfType<ViewResult>()
        .Which.ViewData.Model.Should().BeOfType<ProductIndexViewModel>()
        .Which.Products.Count.Should().Be(20);
}
```

For the assertion I use FluentAssertions and firstly I use the `BeOfType<>()` method to ensure that the result of the controller is of type `ViewResult`:

```csharp
actionResult.Should().BeOfType<ViewResult>()
```

FluentAssertions uses a Fluent API, which means I can chain methods and properties together. The `BeOfType<ViewResult>()` call returns me the action result cast as a `ViewResult`, so I then chain together some more assertions to ensure that the model is of type `ProductIndexViewModel`:

``` csharp
.Which.ViewData.Model.Should().BeOfType<ProductIndexViewModel>()
```

This of course returns me the instance of the model cast correctly as `ProductIndexViewModel`, onto which I can chain one last assertion to ensure that the `Count` of the `Products` property is 20:

``` csharp
.Which.Products.Count.Should().Be(20);
``` 

So the complete assertion firstly checks that the action result is of type `ViewResult`. It then checks that the model for the view is of type `ProductIndexViewModel`, and finally it checks that the number of products assigned to the `ProductIndexViewModel` instance is 20 - so in other words all the products.

> For more information about these particular assertions, you can look at [documentation for FluentAssertions](https://github.com/dennisdoomen/fluentassertions/wiki#basic-assertions) under the "Basic assertions" section.

For the second unit test I pass `false` as the parameter, so in this case I want all 14 products returned. Remember that when I created my test data, I made every third product discontinued, so since I created 20 products I will have 14 products which are not discontinued, and 6 products which are.

So this is the code for that second unit test:

``` csharp
[Fact]
public void IndexShouldNotIncludeDiscontinued()
{
    // Arrange
    var dbContext = serviceProvider.GetRequiredService<ApplicationDbContext>();
    CreateTestData(dbContext);
    var controller = new ProductsController(dbContext);

    // Act
    var actionResult = controller.Index(false);

    // Assert
    actionResult.Should().BeOfType<ViewResult>()
        .Which.ViewData.Model.Should().BeOfType<ProductIndexViewModel>()
        .Which.Products.Count.Should().Be(14);
}
```

## Testing the Details action

For the details action I will do three unit tests. 

The first unit test ensures that the action returns an HTTP 400 (Bad Request) status when no `id` parameter is passed:

``` csharp
[Fact]
public void DetailsShouldReturnBadRequest()
{
    // Arrange
    var dbContext = serviceProvider.GetRequiredService<ApplicationDbContext>();
    CreateTestData(dbContext);
    var controller = new ProductsController(dbContext);

    // Act
    var actionResult = controller.Details(null);

    // Assert
    actionResult.Should().BeOfType<BadRequestResult>();
}
```

The second test checks that the action returns an HTTP 404 (Not Found) when I pass in a non existing product. Remember that when I created the test data I only created 20 products, and the IDs of the products started at 1 and incremented sequentially, so a product with the ID of 21 should not exist:

``` csharp
[Fact]
public void DetailsShouldReturnNotFound()
{
    // Arrange
    var dbContext = serviceProvider.GetRequiredService<ApplicationDbContext>();
    CreateTestData(dbContext);
    var controller = new ProductsController(dbContext);

    // Act
    var actionResult = controller.Details(21);

    // Assert
    actionResult.Should().BeOfType<HttpNotFoundResult>();
}
```

For the last unit test I will pass in a valid product ID. I will then check that the action result is of type `ViewResult`, that the model of that view result is of type `Product` and that the `Id` property of that product has a value of 1:

``` csharp
[Fact]
public void DetailsShouldReturnCorrectProduct()
{
    // Arrange
    var dbContext = serviceProvider.GetRequiredService<ApplicationDbContext>();
    CreateTestData(dbContext);
    var controller = new ProductsController(dbContext);

    // Act
    var actionResult = controller.Details(1);

    // Assert
    actionResult.Should().BeOfType<ViewResult>()
        .Which.ViewData.Model.Should().BeOfType<Product>()
        .Which.Id.Should().Be(1);
}
```

## Conclusion 

In this blog post I demonstrated how you can use the new In-Mmeory database in Entity Framework 7 to easily unit test your applications. Because it uses an in memory data store it means that the "database" can be constructed quickly and your actual unit tests can therefore still run very quickly.

## References

* My previous blog post on [Unit Testing Controllers in ASP.NET 5 (MVC 6)](http://www.jerriepelser.com/blog/unit-testing-controllers-aspnet5)
* Dave Paquette's blog post on [Realistic Sample Data with GenFu](http://www.davepaquette.com/archive/2015/11/15/realistic-sample-data-with-genfu.aspx)
* The XUnit documentation on [Getting Started with xUnit.net (DNX / ASP.NET 5)](https://xunit.github.io/docs/getting-started-dnx.html)
* The EF 7 documentation on [adding EF7 to and ASP.NET 5 Application and creating a new database](http://docs.efproject.net/en/latest/getting-started/aspnet5/new-db.html)
* The [ASP.NET 5 Music Store App](https://github.com/aspnet/MusicStore), and specifically the [test project](https://github.com/aspnet/MusicStore/tree/dev/test/MusicStore.Test). I used this as a reference for understanding how to use the In-Memory database in unit tests.
