---
date: 2015-09-01T00:00:00Z
description: |
  Shows a simple technique you can use to allow users to quickly add records by selecting a checkbox that indicates that they want to add another record after saving.
tags:
- asp.net
- asp.net mvc
title: Allowing users to quicky add records with an 'Add Another' checkbox
url: /blog/allow-users-rapid-cature-with-add-another-checkbox/
---

## Introduction

In a typical web application that performs CRUD actions, the normal workflow for adding a new record is that a user will be presented with a list of records and will then click on an "Add" button to add a new record. This will then navigate them away to a separate web pages where they can add the details for the new record and click on a Save button to add the new record to the database. 

The application will also usually send them back to the list page, so if they want to add a new record they will have to click on the "Add" button again, and so the cycle continues. This can very quickly become cumbersome and frustrating to the users.

In this blog post I will show you an alternate way in which the users can indicate that their intention is to add a new record after saving the current one, so they will immediately be presented with a new blank screen to add the new record after saving, instead of being navigated away to the list again.

## The default behaviour

For this blog post I have create a very basic `Customer` class as well as a database context:

``` csharp
public class Customer
{
    [Key]
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public int Id { get; set; }

    [Required]
    public string FirstName { get; set; }

    [Required]
    public string LastName { get; set; }
}

public class DemoContext : DbContext
{
    public DbSet<Customer> Customers { get; set; }

    public DemoContext()
        : base("DefaultConnection")
    {
    }
}
```

To keep things simple I have created a `CustomersController` class and associated views using the built-in Visual Studio scaffolding. So it created a controller for me that allows me for view a list fo customers, create a new customer, edit a customer, view a customer's details and delete a customer.

Let's look at the current flow for creating a new customer. First we can view the list of customers:

![](/assets/images/2015-09-01-allow-users-rapid-cature-with-add-another-checkbox/customer-list.png)

By clicking on the "Create New" link we are navigated to a new screen where we can complete the customer's information and click on "Create":

![](/assets/images/2015-09-01-allow-users-rapid-cature-with-add-another-checkbox/customer-create.png)

After creating the new customer we are navigated back to the list of customers:

![](/assets/images/2015-09-01-allow-users-rapid-cature-with-add-another-checkbox/customer-list-2.png)

## Changing it to allow for rapid data capture

The first thing I want to do is to add a view model and use that in the Create screen, instead of using the actual Entity Framework model. So add a new class called `CreateCustomerViewModel`:

``` csharp
public class CreateCustomerViewModel
{
    [Required]
    public string FirstName { get; set; }

    [Required]
    public string LastName { get; set; }
}
```

and change `\Views\Customers\Create.cshtml` to use that as a model instead:

```html
@model RapidDataCapture.ViewModels.CreateCustomerViewModel
```

Next head to `CustomersController` and change the current implementation of `Create()` to pass a new instance of `CreateCustomerViewModel`. So it changes from this:

``` csharp
public ActionResult Create()
{
    return View();
}
```

to this:

```csharp
public ActionResult Create()
{
    return View(new CreateCustomerViewModel());
}
```

Why did I do this? Because passing null models to a view is a bad practice. Read the section "Avoid passing null models to views" in [this blog post](http://blogs.msdn.com/b/marcinon/archive/2011/02/07/mvc-performance-tips.aspx).

Next up we need to change the current method that gets invoked on an HTTP POST. The existing method looks like this:

``` csharp
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<ActionResult> Create([Bind(Include = "Id,FirstName,LastName")] Customer customer)
{
    if (ModelState.IsValid)
    {
        _dbContext.Customers.Add(customer);
        await _dbContext.SaveChangesAsync();
        return RedirectToAction("Index");
    }

    return View(customer);
}
```

We want to change the parameter from `Customer` to `CreateCustomerViewModel`, and since we are now using a view model with the exact properties we are interested in there is no more need for the `Bind` attribute. 

After fixes it now looks like this:

``` csharp
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<ActionResult> Create(CreateCustomerViewModel vm)
{
    if (ModelState.IsValid)
    {
        _dbContext.Customers.Add(new Customer
        {
            FirstName = vm.FirstName,
            LastName = vm.LastName
        });

        await _dbContext.SaveChangesAsync();

        return RedirectToAction("Index");
    }

    return View(vm);
}
```

Ok, now the groundwork is laid for the checkbox we want to add to allow users to indicated they want to keep adding records. 

Next I update my view model to add a new property called `CreateAnother`:

```csharp
public class CreateCustomerViewModel
{
    [Required]
    public string FirstName { get; set; }

    [Required]
    public string LastName { get; set; }

	[Display(Name = "Create another customer after saving")]
    public bool CreateAnother { get; set; }
}
```

And add an input field for that on the form after the "Create" button:

``` html
<div class="form-horizontal">
    ...

    <div class="form-group">
        <div class="col-md-offset-2 col-md-10">
            <input type="submit" value="Create" class="btn btn-default"/>
            &nbsp;
            @Html.CheckBoxFor(model => model.CreateAnother)
            @Html.LabelFor(model => model.CreateAnother)
        </div>
    </div>
</div>
```

Next up I need to do is to change my current `Create(CreateCustomerViewModel vm)` method to check whether the "Create another customer after saving" checkbox was ticked. If it was I redirect back to the `Create` action to add another, otherwise I redirect back to the `Index` action.  

``` csharp
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<ActionResult> Create(CreateCustomerViewModel vm)
{
    if (ModelState.IsValid)
    {
        _dbContext.Customers.Add(new Customer
        {
            FirstName = vm.FirstName,
            LastName = vm.LastName
        });

        await _dbContext.SaveChangesAsync();

        if (vm.CreateAnother)
            return RedirectToAction("Create", new {CreateAnother = true});
        else
            return RedirectToAction("Index");
    }

    return View(vm);
}
```

Notice that I pass a `CreateAnother` route value to the `Create` action, so I will need to change the `Create` action that is invoked on an HTTP GET to take that parameter and pass its value along to the view model:

``` csharp
public ActionResult Create(bool createAnother = false)
{
    return View(new CreateCustomerViewModel()
    {
        CreateAnother = createAnother
    });
}
```

By default the value of the `createAnother` parameter is false, so the checkbox is not selected.

Now when I run the application again and click on the "Create New" link on the customer list, I am taken to the customer create screen. Notice the new "Create another customer after saving" checkbox at the bottom:

![](/assets/images/2015-09-01-allow-users-rapid-cature-with-add-another-checkbox/customer-create-2.png)

Complete the form and select the checkbox:

![](/assets/images/2015-09-01-allow-users-rapid-cature-with-add-another-checkbox/customer-create-3.png)

Now when you click in the "Create" button you will be redirected back to the Create screen where you can immediately add a new customer:

![](/assets/images/2015-09-01-allow-users-rapid-cature-with-add-another-checkbox/customer-create-4.png)

## Conclusion

In this blog post I showed how a very small change with minimal effort from you as a developer can make a big impact for the end user by making their life a little bit easier and less frustrating.