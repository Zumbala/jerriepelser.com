---
date: 2015-07-07T00:00:00Z
description: |
  Shows how you can easily create test data with NBuilder and Faker as part of your Seed data generation process for Entity Framework.
tags:
- aspnet
- aspnet mvc
- nbuilder
- faker
title: Create test data with NBuilder and Faker
url: /blog/creating-test-data-with-nbuilder-and-faker/
---

For many web applications you may want to generate some initial, semi-realistic test data for your development and testing environments. Entity Framework provides a handy mechanism through which you can achieve this, namely seed data. 

You can read Mike Wasson's blog post on the ASP.NET website for some background on [how to use create seed data with Entity Framework Code-First migrations](http://www.asp.net/web-api/overview/data/using-web-api-with-entity-framework/part-3).

The Seed mechanism is handy for populating the database, but how do you create the actual test data? You can create the records by hand, perhaps import from an external file, or just about any other way you can think of.

When it comes to generating the data, I personally make use of two 3rd party libraries, namely [NBuilder](https://www.nuget.org/packages/NBuilder) and [Faker.NET](https://www.nuget.org/packages/Faker.Net). Both these libraries have not been updated for a long time, but they still work just fine.

Let's look at an example of how to use this.

## Setting things up

For this blog post I created a new ASP.NET web application and enabled Entity Framework migrations. I also created some models which represent a customer with orders. Here is what the customer and order models and my database context looks like:

``` csharp
public class Customer
{
    public string EmailAddress { get; set; }
    public string FirstName { get; set; }
    public int Id { get; set; }
    public string LastName { get; set; }
    public string TelephoneNumber { get; set; }
}

public class Order
{
    public Customer Customer { get; set; }
    public int Id { get; set; }
    public DateTime OrderDate { get; set; }
}

public class Product
{
    public string Description { get; set; }
    public int Id { get; set; }
    public string Name { get; set; }
    public float Price { get; set; }
}

public class OrderItem
{
    public int Id { get; set; }
    public Order Order { get; set; }
    public Product Product { get; set; }
    public int Quantity { get; set; }
}

public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
{
    public virtual IDbSet<Customer> Customers { get; set; }
    public virtual IDbSet<OrderItem> OrderItems { get; set; }
    public virtual IDbSet<Order> Orders { get; set; }
    public virtual IDbSet<Product> Products { get; set; }

    public ApplicationDbContext()
        : base("DefaultConnection", false)
    {
    }

    public static ApplicationDbContext Create()
    {
        return new ApplicationDbContext();
    }
}
```

Next we will need to add the NBuilder and Faker.NET Nuget packages to our application:

``` text
Install-Package NBuilder
Install-Package Faker.Net
```

## Using NBuilder to create objects

So Let's get going with NBuilder to create some test objects. To generate a list of customers I start off by creating a `Builder<>` of type `Customer` and specifying that we want to create a list of 100 items. After configuring how the objects should be generated you simply call the `Build()` method to generate the actual objects.

I suggest you also [look at the NBuilder documentation](https://code.google.com/p/nbuilder/wiki/Overview_HowTo) to understand a little bit more about how to generate objects.

Finally, to insert the objects into the database, we call the `AddOrUpdate` extension method on the Customers `DBSet` of the database context.  

``` csharp
protected override void Seed(Models.ApplicationDbContext context)
{
    var customers = Builder<Customer>.CreateListOfSize(100)
        .Build();

    context.Customers.AddOrUpdate(c => c.Id, customers.ToArray());
}
```

Simple as that. When running the database migration we get 100 records generated and inserted into the Customers table:

![](/assets/images/creating-test-data-with-nbuilder-and-faker/simple-100-customers.png)

## Using Faker to make it realistic

The data generated in the example above is great, but unfortunately does not look very realistic. It would be great to have something which looks like actual names, email addresses and telephone numbers. This is where Faker.NET comes in. 

Faker.NET is a port of a popular Ruby project called [faker](http://faker.rubyforge.org/). It allows you to generate a wide range of realistic looking data, including names, addresses, telephone numbers, internet domains, sentences, etc.

To use it we need to use the correct Faker generator for each individual field of the `Customer` class. Using NBuilder's fluent API we simply need to specify which items in the list we want to operate on, and then then specify for that sub-segment of items the values for each individual field.

In this case we want to operate on all the items, so I specify that with the `All()` extension method, and then I use the `With()` extension method to specify the values for the individual fields. The `With()` methods takes a function expression which allows me to return any value. So you can return a hard coded value for a field, or in this case I will return a value which is generated by the Faker library.  

``` csharp
protected override void Seed(Models.ApplicationDbContext context)
{
    var customers = Builder<Customer>.CreateListOfSize(100)
        .All()
            .With(c => c.FirstName = Faker.Name.First())
            .With(c => c.LastName = Faker.Name.Last())
            .With(c => c.EmailAddress = Faker.Internet.Email())
            .With(c => c.TelephoneNumber = Faker.Phone.Number())
        .Build();

    context.Customers.AddOrUpdate(c => c.Id, customers.ToArray());
}
```

Now when we run the migration, you will notice that we get a little bit more realistic values:

![](/assets/images/creating-test-data-with-nbuilder-and-faker/realistic-customers.png)

## Operating on a subset of data

In the example above we operated on all the objects when specifying the values for the various properties, but still this is not necessarily representative of a real-life scenario. In real life not all people have email addresses and phone numbers, or may be simply unwilling to supply those. To operate on a specify subset of data in NBuilder we can use one of the various other extension methods available.

We can use the extension method `TheFirst()` for example to operate on the first x number of objects. So to specify that we want to operate on the first 10 objects only, we can specify `TheFirst(10)` and then specify the various property values for the first 10 objects. Other methods are available such as `TheNext()`, `TheLast()` and `ThePrevious()`. You can also use `Section()` to operate on a specify section of the objects (by specifying a start and end index). 

Once again I suggest you [look at the NBuilder documentation](https://code.google.com/p/nbuilder/wiki/Overview_HowTo) on what is available to you. Once thing to note is that the methods specified in the documentation is out of date. It specifies `WhereTheFirst()` for example instead of `TheFirst()`. The methods are marked as obsolete however so if you look at the intellisense in Visual Studio it will tell you which method to use instead:

![](/assets/images/creating-test-data-with-nbuilder-and-faker/obsolete-intellisense.png)

I am going to use the `Random()` method to specify that I want to operate on a random number of objects. So I will say that only a random 70 customers have email addresses. The only thing to note is that I need to specify that by default all customers have no email address, i.e. `With(c => c.EmailAddress = null)` otherwise it will still generate email address along the pattern of "EmailAddress1" for those other 30 customers.

``` csharp
protected override void Seed(Models.ApplicationDbContext context)
{
    var customers = Builder<Customer>.CreateListOfSize(100)
        .All()
            .With(c => c.FirstName = Faker.Name.First())
            .With(c => c.LastName = Faker.Name.Last())
            .With(c => c.EmailAddress = null)
            .With(c => c.TelephoneNumber = Faker.Phone.Number())
        .Random(70)
            .With(c => c.EmailAddress = Faker.Internet.Email())
        .Build();
            
    context.Customers.AddOrUpdate(c => c.Id, customers.ToArray());
}
```

Running the migration you will notice that we now have some random customers with no email address:

![](/assets/images/creating-test-data-with-nbuilder-and-faker/random-email-addresses.png)

## Generating the products

Generating the products is also fairly simple. Unfortunately there is no way with Faker to create product names, so for the names of the I will concatenate the word "Product " and the Id of the product, so I will end up with names like "Product 1", "Product 2", etc. I could also not have specified anything and it would have generated the product names "Name1", "Name2", etc.

For the description I use the Faker.NET Lorem generator to generate a [Lorem Ipsum](https://en.wikipedia.org/wiki/Lorem_ipsum) paragraph of text.

For the price of the product I want to generate a value between 50 and 500, so I create an instance of the NBuilder `RandomGenerator` helper class, and then simply call the `Next()` method, specifying a range from 50 to 500 to generate a next random value inside that range.

``` csharp
protected override void Seed(Models.ApplicationDbContext context)
{
    // Generate customers
	...
    
    // Generate products
	var priceGenerator = new RandomGenerator();
	
	var products = Builder<Product>.CreateListOfSize(50)
	    .All()
	        .With(p => p.Name = "Product " + p.Id.ToString())
	        .With(p => p.Description = Faker.Lorem.Paragraph())
	        .With(p => p.Price = priceGenerator.Next(50, 500))
	    .Build();
	
	context.Products.AddOrUpdate(p => p.Id, products.ToArray());
}
```

And this generates the following list of products:

![](/assets/images/creating-test-data-with-nbuilder-and-faker/products.png)

## Generating the orders

Generating the orders become a little bit more interesting. For the order date I want to generate a random date up to 100 days in the past. To do that I once again instantiate an instance of the `RandomGenerator` class and then use that to subtract a random number of days between 1 and 100 from today's date.

For the customers I would like to pick a customer from the already generated list of customers. To do this NBuilder has a `Pick` class, which allows me to pick random item(s) from a secondary list. You can use either the `RandomItemFrom()` method to pick a single item, or use the `UniqueRandomList()` method to pick a list of items from the secondary list.

I just want to select a single `Customer` for each `Order`, so I use the `RandomItemFrom()` method:  

``` csharp
protected override void Seed(Models.ApplicationDbContext context)
{
    // Generate customers
	...
    
    // Generate products
	...

    // Generate the orders
    var daysGenerator = new RandomGenerator();

    var orders = Builder<Order>.CreateListOfSize(200)
        .All()
            .With(o => o.Customer = Pick<Customer>.RandomItemFrom(customers))
            .With(o => o.OrderDate = DateTime.Now.AddDays(-daysGenerator.Next(1, 100)))
        .Build();

    context.Orders.AddOrUpdate(o => o.Id, orders.ToArray());
}
```

This will generate 200 orders by picking a random customer from the list of already generated customers, so some of my 100 customers may end up with no orders, and some may end up with more than one order.
 
Running the migrations, this is what I end up with for my Orders table:

![](/assets/images/creating-test-data-with-nbuilder-and-faker/orders.png)

## Generating the order items

For the order items I want to generate a random number of order items for each order. I will therefore iterate through the list of orders which was generated before and simply generate a random number of items by using a `RandomGenerator` again and generating a value of between 1 and 10 to generate that number of order items for the specific order.

To generate the actual property values for the order, I simply use all of the techniques demonstrated before. For the `Order` property I use the current order I am iterating over. For the `Product` I pick a random item from the previous list of generated products, and for the quantity I generate a random quantity between 1 and 10.

This is what my final `Seed()` method looks like:

``` csharp
protected override void Seed(Models.ApplicationDbContext context)
{
    // Generate customers
    var customers = Builder<Customer>.CreateListOfSize(100)
        .All()
            .With(c => c.FirstName = Faker.Name.First())
            .With(c => c.LastName = Faker.Name.Last())
            .With(c => c.EmailAddress = null)
            .With(c => c.TelephoneNumber = Faker.Phone.Number())
        .Random(70)
            .With(c => c.EmailAddress = Faker.Internet.Email())
        .Build();
            
    context.Customers.AddOrUpdate(c => c.Id, customers.ToArray());

    // Generate products
    var priceGenerator = new RandomGenerator();

    var products = Builder<Product>.CreateListOfSize(50)
        .All()
            .With(p => p.Name = "Product " + p.Id.ToString())
            .With(p => p.Description = Faker.Lorem.Paragraph())
            .With(p => p.Price = priceGenerator.Next(50, 500))
        .Build();

    context.Products.AddOrUpdate(p => p.Id, products.ToArray());

    // Generate the orders
    var daysGenerator = new RandomGenerator();

    var orders = Builder<Order>.CreateListOfSize(200)
        .All()
            .With(o => o.Customer = Pick<Customer>.RandomItemFrom(customers))
            .With(o => o.OrderDate = DateTime.Now.AddDays(-daysGenerator.Next(1, 100)))
        .Build();

    context.Orders.AddOrUpdate(o => o.Id, orders.ToArray());

    // Generate order items
    var itemCountGenerator = new RandomGenerator();
    var quantityGenerator = new RandomGenerator();

    orders.ForEach(o =>
                    {
                        var orderItems = Builder<OrderItem>.CreateListOfSize(itemCountGenerator.Next(1, 10))
                            .All()
                                .With(oi => oi.Order = o)
                                .With(oi => oi.Product = Pick<Product>.RandomItemFrom(products))
                                .With(oi => oi.Quantity = quantityGenerator.Next(1, 10))
                            .Build();

                        context.OrderItems.AddOrUpdate(oi => oi.Id, orderItems.ToArray());
                    });
}
```

## Conclusion

In this blog post I demonstrated how you can use the combination of [NBuilder](https://www.nuget.org/packages/NBuilder) and [Faker.NET](https://www.nuget.org/packages/Faker.Net) to generate realistic looking data for your development and test environments.

The `Seed()` method will run everytime you run the database migrations, so you should propbably add some sort of switch which reads a value from an environment variable to disable the generation of test data for your production environment.
