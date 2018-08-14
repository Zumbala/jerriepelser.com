---
title: "Deleting records in ASP.NET Core using AJAX"
description:
  You can delete records and update the user interface using Ajax, which give a much better user experience that refreshing the entire page.
tags:
- asp.net core
- jquery
- ajax
---

## Introduction

In my [introductory post](/blog/you-might-not-need-angular-introduction/) I mentioned that you could achieve a lot of the interactivity of client-side frameworks, such as Angular, by using tried and tested techniques such as Ajax. This allows you to selectively apply these techniques to an existing code-base without having to go all-in on one of these modern client-side frameworks - which very often bring a lot of complexity with them as well.

In this blog post, we will start off with a reasonably simple example - that of deleting a record without a full page refresh.

## Current scenario

Let's say we have a grid with a list of customers being displayed. The user can delete any customer by clicking on the **Delete** button next to a particular customer. This will do an HTTP POST to a `Delete` action which will remove the customer from the database and redirect the user back to the list of customers.

Here is the controller for this current scenario:

```csharp
public class Scenario1Controller : Controller
{
    private readonly Database _database;

    public Scenario1Controller(Database database)
    {
        _database = database;
    }

    public IActionResult Index()
    {
        return View(_database.Customers);
    }

    [HttpPost]
    public IActionResult Delete(int id)
    {
        _database.Customers.RemoveAll(customer => customer.Id == id);

        return RedirectToAction("Index");
    }
}
```

Here is the relevant part from the view returned from the `Index` action.

```html
<ul class="list-group list-group-lg list-group-flush list my--4">
    @foreach(var customer in Model)
    {
    <li class="list-group-item px-0">
        <div class="row align-items-center">
            <div class="col-auto">
                <span class="avatar avatar-lg">
                    <img src="@customer.Avatar" alt="" class="avatar-img">
                </span>
            </div>
            <div class="col ml--2">
                <h4 class="card-title mb-1">
                    <a href="#!">@GetCustomerFullName(customer)</a>
                </h4>
                <p class="card-text small text-muted mb-1">
                    @customer.EmailAddress
                </p>
            </div>
            <div class="col-auto">
                <form asp-action="Delete" asp-route-id="@customer.Id">
                    <button type="submit" class="btn btn-sm btn-danger d-none d-md-inline-block">
                        Delete
                    </button>
                </form>
            </div>
        </div>
    </li>
    }
</ul>
```

The database used by in sample is a singleton, in-memory list of customers with realistic data being generated using the [Bogus](https://github.com/bchavez/Bogus) library. You can look at the source code accompanying the blog post if you want to see how that is done.

Here is a short animation of this in action. Notice the full page refresh each time a customer is deleted.

<img class="gifplayer" 
    src="/images/blog/2018-08-14-deleting-records-aspnet-core-ajax/delete-record-postback.png"
    data-gif="/images/blog/2018-08-14-deleting-records-aspnet-core-ajax/delete-record-postback.gif"
    style="width: 100%;"
    alt="Delete a record using postbacks"
    />

## Improving the experience with Ajax

Now, let's look at how we can improve the experience using Ajax and the [jQuery Unobtrusive Ajax library](https://github.com/aspnet/jquery-ajax-unobtrusive).

First, you will need to add the jQuery Unobtrusive Ajax library to your project. I downloaded the JavaScript file, added it to my project and referenced it in my `_Layout` view.

Next, we'll need to update the `<form>` tag we declared in the View which calls the `Delete` handler to add a `data-ajax` attribute. This tells jQuery Unobtrusive to intercept that postback and make an Ajax call instead.

```html
<form asp-action="Delete" asp-route-id="@customer.Id" data-ajax="true">
    <button type="submit" class="btn btn-sm btn-danger d-none d-md-inline-block">
        Delete
    </button>
</form>
```

We will also need to update `Delete` action. Previously, it redirected the user to the `Index` action after a record was deleted. Since this action is now invoked using Ajax, we do not have to do the redirect anymore - or return any content for that matter.

```csharp
[HttpPost]
public IActionResult Delete(int id)
{
    _database.Customers.RemoveAll(customer => customer.Id == id);

    return NoContent();
}
```

At this point, the customer will be deleted using Ajax, but the list of customers will not be updated to reflect the fact that the customer was deleted. To do this, we will need to manually delete the `<li>` element containing the customer information after the Ajax call has completed.

We need some sort of callback after the Ajax call has completed so we can handle this deletion. The jQuery Unobtrusive Ajax library allows us to do this by specifying a function to be called using the `data-ajax-success` attribute.

Let's update the `<form>` to add this attribute which specifies a `deleteItem` function, passing the form element itself as a parameter:

```html
<form asp-action="Delete" asp-route-id="@customer.Id" data-ajax="true" data-ajax-success="deleteItem(this)">
    <button type="submit" class="btn btn-sm btn-danger d-none d-md-inline-block">
        Delete
    </button>
</form>
```

The delete item itself is pretty straight-forward. We traverse the DOM tree for the ancestors of the `<form>` and look for that `<li>` parent element. Once we find it, we remove it:

```html
<script>
    function deleteItem(form) {
        $(form).parents('li').remove();
    }
</script>
```

With that in place, here is another animation demonstrating this improved version. Notice the network traffic in the developer tools window which indicates the requests initiated by clicking the Delete button as [XHR requests](https://en.wikipedia.org/wiki/XMLHttpRequest) (i.e. Ajax calls):

<img class="gifplayer" 
    src="/images/blog/2018-08-14-deleting-records-aspnet-core-ajax/delete-record-ajax.png"
    data-gif="/images/blog/2018-08-14-deleting-records-aspnet-core-ajax/delete-record-ajax.gif"
    style="width: 100%;"
    alt="Delete a record using Ajax"
    />

## Conclusion

In this blog post, I showed how you could create a better user experience using Ajax to delete a customer record. It is a relatively simple process which can be added to an existing code base and does not require you to go all-in on any of the modern JavaScript frameworks.

Source code is available at [https://github.com/jerriepelser-blog/aspnetcore-ajax](https://github.com/jerriepelser-blog/aspnetcore-ajax).