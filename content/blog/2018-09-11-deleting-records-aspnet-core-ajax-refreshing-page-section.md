---
title: "Deleting records in ASP.NET Core using AJAX with partial page refresh"
description:
  This follow-up from my previous blog post on the topic demonstrates how to refresh an entire section of a page when deleting a record with AJAX.
tags:
- asp.net core
- jquery
- ajax
---

## Introduction

This is part of a series on using the jQuery Unobtrusive Ajax library in ASP.NET Core applications. Here are this list of posts:

* [You might not need Angular](/blog/you-might-not-need-angular-introduction/)
* [Deleting records in ASP.NET Core using AJAX](/blog/deleting-records-aspnet-core-ajax/)
* Deleting records in ASP.NET Core using AJAX with partial page refresh (this blog post)

In the [previous blog post in this series](/blog/deleting-records-aspnet-core-ajax/), I demonstrated how you can delete a record with Ajax and then remove the row from an HTML table with JavaScript.

This works fine, but there are scenarios where you may want to display different content depending on whether the page contains records or not. In the current scenario, when there are no customers to display, the page will just display an empty grid:

![Empty grid with no customers](/images/blog/2018-09-11-deleting-records-aspnet-core-ajax-refreshing-page-section/empty-grid.png)

It would be better to display an empty state which informs the user that there are no customers and allow them to click on a button to add a new customer:

![Empty state with status message and button](/images/blog/2018-09-11-deleting-records-aspnet-core-ajax-refreshing-page-section/empty-state.png)

Since the logic deciding whether to render the grid or an empty state happens on the server, the current approach of simply deleting the rows from the table using JavaScript will not work. Instead, we will need to render that section of the page on the server every time a customer is deleted, and then update the HTML on the client side.

## Changing the view

The first thing to do is to extract the logic that renders the card containing the customer grid to a partial view called `_CustomerList.cshtml`. Note that the partial view was also update with logic to render different content depending on whether there are any customers or not.

If there are no customers, we will render a message to the user and display a button they can click to add a new customer. Otherwise, the list of customers will be displayed, same as [last time](/blog/deleting-records-aspnet-core-ajax/).

The other major change is that we removed the `data-ajax-success` attribute from the `form` and replaced that with a `data-ajax-update` attribute. This attribute contains the reference to the HTML element which content should be updated with the content returned from the Ajax call. In this case, I state that the content of the `div` with the ID `CustomerList` should be updated (we'll add this code in a moment).

```html
<!-- _CustomerList.cshtml -->
@using DemoApp.Data
@model List<Customer>

<div class="card mt-5">
    <div class="card-header">
        <div class="row align-items-center">
            <div class="col">
                <h4>
                    Customers
                </h4>
            </div>
        </div>
    </div>
    <div class="card-body">
        @if (Model.Count == 0)
        {
            <div class="row justify-content-center text-center">
                <div class="col-12 col-md-10 mt-5">

                    <i class="fa fa-user fa-5x mb-5"></i>

                    <h2>
                        No Customers
                    </h2>

                    <p class="text-muted">
                        You do not have any customers yet. Add a new customer by clicking the <strong>Add Customer</strong> button below.
                    </p>
                    
                    <a class="btn btn-lg btn-primary d-none d-md-inline-block mt-3" href="#">Add Customer</a>
                </div>
            </div> <!-- / .row -->
        }
        else
        {
            <ul class="list-group list-group-lg list-group-flush list my--4">
                @foreach (var customer in Model)
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
                                <form asp-action="Delete" asp-route-id="@customer.Id" data-ajax="true" data-ajax-update="#CustomerList">
                                    <button type="submit" class="btn btn-sm btn-danger d-none d-md-inline-block">
                                        Delete
                                    </button>
                                </form>
                            </div>
                        </div>
                    </li>
                }
            </ul>
        }
    </div>
</div>

@functions {
    string GetCustomerFullName(Customer customer)
    {
        return $"{customer.FirstName} {customer.LastName}";
    }
} 
```

Let's update the main view to now call the partial view. Also not that I am rendering the partial view inside a `div` called `CustomerList`. As stated above, the content of this `div` will be replaced with the content that is returned from the server when a user is deleted because I supplied a reference to this in the partial view in the `data-ajax-update` attribute of the `form`.

```html
@using DemoApp.Data
@model List<Customer>

@{
    ViewBag.Title = "Customers";
}

<div class="container">
    <div class="row">
        <div class="col">
            <div id="CustomerList">
                <partial name="_CustomerList" model="Model"/>
            </div>
        </div>
    </div>
</div>
```

## Updating the controller

We will also need to update the controller. Previously, when a user deleted a customer we simply returned no content. This time around, we will render the `_CustomerList` partial and return the result of that.

```cs
public class Scenario2Controller : Controller
{
    private readonly Database _database;

    public Scenario2Controller(Database database)
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

        return PartialView("_CustomerList", _database.Customers);
    }
}
```

## Trying it out

With all of that in place, it is time to try it out. In the small screencast below you will notice me deleting customers, and once the last customer is deleted, the page will display the "empty state" content correctly:

<img class="gifplayer" 
    src="/images/blog/2018-09-11-deleting-records-aspnet-core-ajax-refreshing-page-section/delete-record.png"
    data-gif="/images/blog/2018-09-11-deleting-records-aspnet-core-ajax-refreshing-page-section/delete-record.gif"
    style="width: 100%;"
    alt="Delete a record with partial page refresh"
    />

## Conclusion

In this blog post, I showed how you can return a partial result when deleting a record using Ajax. It is a relatively simple process which can be added to an existing code base and does not require you to go all-in on any of the modern JavaScript frameworks.

Source code is available at [https://github.com/jerriepelser-blog/aspnetcore-ajax](https://github.com/jerriepelser-blog/aspnetcore-ajax). Have a look at the **Scenario 2** controller and views.