---
date: 2015-10-13T00:00:00Z
description: |
  One of the new things I had to figure out in doing unit testing in ASP.NET 5 was how to mock the necessary infrastructure to test my controllers. In this blog post I show a few techniques you can use.
tags:
- asp.net
- asp.net 5
- asp.net mvc
- unit testing
title: Unit testing controllers in ASP.NET 5 (MVC 6)
url: /blog/unit-testing-controllers-aspnet5/
---

## Introduction

Part of my journey in working with ASP.NET is figuring out how to replace the way in which I did things before with a more appropriate way of doing it in ASP.NET 5. One of these is figuring out how to test my controllers.

Previously in ASP.NET MVC 5 I used the awesome [FluentMVCTesting](https://github.com/TestStack/TestStack.FluentMVCTesting) library, but currently it does not support ASP.NET 5, so I had to revert back to the old ways of doing things.

In ASP.NET 5 you can use POCO classes as controllers (see [POCO controllers in ASP.NET vNext](http://www.strathweb.com/2014/06/poco-controllers-asp-net-vnext/) and [How ASP.NET MVC 6 discovers controllers?](http://www.strathweb.com/2015/04/asp-net-mvc-6-discovers-controllers/)), but right now all of my controllers still inherit from the Controller base class.

## Basics of a controller

The reason I inherit from the base `Controller` class is because it gives me access to a whole bunch of properties such as ModelState, the current user, and perhaps even the underlying request. Yeah, I can use POCO controllers and have the Dependency Injection infrastructure inject everything I require, but this way it makes my life a little bit easier.

At this point I want to point you to the source code for the `Controller` class on Github:
[https://github.com/aspnet/Mvc/blob/dev/src/Microsoft.AspNet.Mvc.ViewFeatures/Controller.cs](https://github.com/aspnet/Mvc/blob/dev/src/Microsoft.AspNet.Mvc.ViewFeatures/Controller.cs)

If you look through that source code you will very quickly notice that one specific object stands out, and that is the `ActionContext`. Most of the other properties gets their values from the `ActionContext` class, and that makes our life a little bit easier since we really only have to supply an instance of ActionContext, and supply mocked objects for some of its properties.

If you look at [the source code for ActionContext](https://github.com/aspnet/Mvc/blob/dev/src/Microsoft.AspNet.Mvc.Abstractions/ActionContext.cs), you will see a lot of comments throughout referring that a certain constructor or property setters are provided "for testing purposes only", so it would seem to me that the intention is that the ActionContext is indeed the object which you should use to set up testing scenarions.    

So in the rest of the blog post we will look at how you would go about setting up a couple of these typical testing scenarios.

## Simulating Model errors

One of the first scenarions I want to demonstrate is to simulate model errors, so let us assume I have an action with the following line code:

``` csharp
public async Task<IActionResult> Create(CustomerCreateModel model)
{
    if (ModelState.IsValid)
    {
        // Create a new customer
		// ... code omitted for brevity....

		// Redirect to the index
        return RedirectToAction("Index");
    }

    return View(model);
}
```

I want to test to ensure that if the model contains errors that the View is returned, otherwise the customer must be saved and the user must be redirected to the Index page.

```csharp
[Fact]
public async Task CreateShouldReturnViewWhenModelHasErrors()
{
    // Arrange
    var controller = new CustomersController();
    controller.ModelState.AddModelError("", "dummy error");

    // Act
    var actionResult = await controller.Create(new CustomerCreateViewModel());

    // Assert
    actionResult.Should().BeOfType<ViewResult>();
}
```

In the code above, the first thing I do is to simulate model errors by simply adding a model error using the following line:

```csharp
controller.ModelState.AddModelError("", "dummy error");
```
 
This means that the `if (ModelState.IsValid)` condition will return a `false` and the code will fall through to returning the Create View again with the current model and will therefore display the validation errors.

To test whether this happen I use the [FluentAssertions](http://www.fluentassertions.com/) library and simply check if the returned action result is of type `ViewResult`:

```csharp
// Assert
actionResult.Should().BeOfType<ViewResult>();
```

And for completeness sake, to test a scenario where there are no model errors I can write the following test to ensure the user is redirect back to the Index action:

```csharp
[Fact]
public async Task CreateShouldReturnViewWhenModelHasErrors()
{
    // Arrange
    var controller = new CustomersController();

    // Act
    var actionResult = await controller.Create(new CustomerCreateViewModel());

    // Assert
    actionResult.Should().BeOfType<RedirectToActionResult>()
        .Which.ActionName.Should().Be("Index");
}
```

## Mocking the current user

Another situation I ran into was that I had to get the information of the current user. So let us say for example that I want to store the details of the current user on a customer to indicate the user who created the customer.

``` csharp
public async Task<IActionResult> Create(CustomerCreateModel model)
{
    if (ModelState.IsValid)
    {
        // Create a new customer
		var customer = new Customer 
		{
			CreatedByUserId = User.GetUserId()
		}

		// Map the rest of the customer properties and save the customer
		// ... code omitted for brevity....

		// Redirect to the index
        return RedirectToAction("Index");
    }

    return View(model);
}
```

If we [look at the code](https://github.com/aspnet/Identity/blob/dev/src/Microsoft.AspNet.Identity/PrincipalExtensions.cs) for the `GetUserId` extension method we will see the following:

```csharp
/// <summary>
/// Returns the User ID claim value if present otherwise returns null.
/// </summary>
/// <param name="principal">The <see cref="ClaimsPrincipal"/> instance this method extends.</param>
/// <returns>The User ID claim value, or null if the claim is not present.</returns>
/// <remarks>The User ID claim is identified by <see cref="ClaimTypes.NameIdentifier"/>.</remarks>
public static string GetUserId(this ClaimsPrincipal principal)
{
    if (principal == null)
    {
        throw new ArgumentNullException(nameof(principal));
    }
    return principal.FindFirstValue(ClaimTypes.NameIdentifier);
}
```

The method is looking for the value the `NameIdentifier` claim type to get the value of User ID. 

That is one part of the puzzle. The other part is about where the controller access the value of the `User` property. For that we [head back to the code for the Controller class](https://github.com/aspnet/Mvc/blob/dev/src/Microsoft.AspNet.Mvc.ViewFeatures/Controller.cs) and see the following:

```csharp
/// <summary>
/// Gets or sets the <see cref="ClaimsPrincipal"/> for user associated with the executing action.
/// </summary>
public ClaimsPrincipal User
{
    get
    {
        return HttpContext?.User;
    }
}
```

And where does HttpContext come from? The ActionContext:

```csharp
/// <summary>
/// Gets the <see cref="Http.HttpContext"/> for the executing action.
/// </summary>
public HttpContext HttpContext
{
    get
    {
        return ActionContext?.HttpContext;
    }
}
```

So as I mentioned earlier, the `ActionContext` seems to be your friend in unit testing. 

To test my controller action above with a specific user Id, I can write the following unit test:

```csharp
[Fact]
public async Task EnsureUserGetsStored()
{
    // Arrange
    var validPrincipal = new ClaimsPrincipal(
        new[]
        {
            new ClaimsIdentity(
                new[] {new Claim(ClaimTypes.NameIdentifier, "MyUserId")})
        });

    var httpContext = Substitute.For<HttpContext>();
    httpContext.User.Returns(validPrincipal);

    var controller = new CustomersController()
    {
        ActionContext = new ActionContext
        {
            HttpContext = httpContext
        }
    };

    // Act 
    var actionResult = await controller.Create(new PostCreateViewModel());

    // Assert
	// do whatever assertions you need to do to ensure user id was set correctly...
}
``` 

The code above does the following

1. Firstly I need to create a `ClaimsPrincipal` instance to which I pass a `ClaimsIdentity`, on which I have set the `NameIdentifier` claim type with the value of the User ID I want to simulate.
2. Next I mock the `HttpContext` class using [NSubstitute](https://github.com/nsubstitute/NSubstitute), and ensure that my mock object returns the principal created above as the value of the `User` property.
3. Then I create an instance of my controller and ensure that for the `ActionContext` property I have created a new `ActionContext` class with the `HttpContext` property set to my mock `HttpContext`.

And that is all I need to do. Now when I run the unit test the User Id will be set to "MyUserId" and I can write appropriate assertions to ensure that the Customer that gets saved to the database has the correct value set for the `CreatedByUserId` property.

## Conclusion

I demonstrated only two scenarios in this blog post, but with this knowledge I think you can pretty much figure out most of the other scenarios you want to test. 

If you get stuck, remember to look at the source code for the [Controller class](https://github.com/aspnet/Mvc/blob/dev/src/Microsoft.AspNet.Mvc.ViewFeatures/Controller.cs), as well as [the source code for ActionContext](https://github.com/aspnet/Mvc/blob/dev/src/Microsoft.AspNet.Mvc.Abstractions/ActionContext.cs).