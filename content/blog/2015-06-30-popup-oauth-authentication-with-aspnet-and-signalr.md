---
date: 2015-06-30T00:00:00Z
description: |
  Shows how you can authenticate a user with an OAuth provider using a popup dialog. Also uses SignalR for callbacks to automatically refresh the page after authentication.
tags:
- aspnet
- signalr
- oauth
title: Popup OAuth authentication with ASP.NET and SignalR
url: /blog/popup-oauth-authentication-with-aspnet-and-signalr/
---

## Introduction

One of the SaaS tools which I like to use for all sorts of small app-to-app integrations is [Zapier](https://zapier.com/). If you have ever used Zapier before you may have noticed that authenticating with one of the many applications with which it integrates does not actually navigate away from the current page that you are on, but instead uses a simple HTML popup dialog to authenticate the user.

For a recent project I was involved in I had to do something similar, so I have extracted the techniques into a demo application and I am writing this blog post to show how easy it is to do something like this. 
  
This blog post builds on a previous blog post I did about [Using the ASP.NET OAuth providers without ASP.NET Identity](http://www.jerriepelser.com/blog/using-aspnet-oauth-providers-without-identity). This example therefore does not use ASP.NET Identity, but the concepts I will demonstrate in this blog post can certainly be applied to a normal ASP.NET MVC application which uses ASP.NET Identity.

## Cleaning up a few things from the previous example

My first task is to clean up some of the code from the previous example to make it a bit more modular, so I extracted the code that retrieves the user profile from GitHub into a `GetUserProfile()` helper method. So now the code for my `HomeController` looks as follows:

``` csharp
public class HomeController : Controller
{
    public async Task<ActionResult> Index()
    {
        var userProfile = await GetUserProfile();

        return View(userProfile);
    }

    public ActionResult AuthorizeGitHub()
    {
        return new ChallengeResult("GitHub", Url.Action("Index"));
    }

    public async Task<UserDisplayModel> GetUserProfile()
    {
        var userDisplayModel = new UserDisplayModel();

        var authenticateResult = await HttpContext.GetOwinContext().Authentication.AuthenticateAsync("ExternalCookie");
        if (authenticateResult != null)
        {
            var tokenClaim = authenticateResult.Identity.Claims.FirstOrDefault(claim => claim.Type == "urn:token:github");
            if (tokenClaim != null)
            {
                var accessToken = tokenClaim.Value;

                var gitHubClient = new GitHubClient(new ProductHeaderValue("OAuthTestClient"));
                gitHubClient.Credentials = new Credentials(accessToken);

                var user = await gitHubClient.User.Current();

                userDisplayModel.AccessToken = accessToken;
                userDisplayModel.Name = user.Name;
                userDisplayModel.AvatarUrl = user.AvatarUrl;
            }
        }
        return userDisplayModel;
    }
}
``` 

I have also cleaned up the view for the home page by extracting the code which displays the user profile into a separate partial view, so my `Index.cshtml` now looks as follows:

``` html
@model OAuthPopupAuthentication.Models.UserDisplayModel
@{
    ViewBag.Title = "Home Page";
}

<div id="user-profile">
    @Html.Partial("_UserProfilePartial", Model)
</div>
```

And the `_UserProfilePartial.cshtml` partial checks to see whether we have an OAuth access token for the user (i.e. the user is autenticated), and if so it displays the user information, Otherwise it displays a button which allows the user to authenticate with GitHub.

``` html
@model OAuthPopupAuthentication.Models.UserDisplayModel

@if (!string.IsNullOrEmpty(Model.AccessToken))
{
    <h2>GitHub Details</h2>
    <p>Access Token: @Model.AccessToken</p>
    <p>Name: @Model.Name</p>
    <img src="@Model.AvatarUrl" style="width: 100px; height: 100px;"/>
}
else
{
    @Html.ActionLink("Authenticate with GitHub", "AuthorizeGitHub", null, new {id = "authenticate-github", @class = "btn btn-primary btn-lg"})
}
``` 

## Doing the authentication via a popup dialog

To perform the OAuth authentication via a popup dialog, I add a simple piece of script to my `Index.cshtml` to bind a jQuery click handler to the authenticate button. When the user clicks on the button, the application will open the `href` of the button in a popup dialog. Remember from the previous example when clicking on that button, the `AuthorizeGitHub` action is executed which will return a `ChallengeResult` which will request the OWIN middleware to authenticate the user with GitHub.

So effectively what will happen is when the user clicks of the button, it will open the GitHub OAuth authentication dialog in a popup window.

Here's the code for my updated `Index.cshtml`:

``` html
@model OAuthPopupAuthentication.Models.UserDisplayModel
@{
    ViewBag.Title = "Home Page";
}

<div id="user-profile">
    @Html.Partial("_UserProfilePartial", Model)
</div>

@section Scripts {

    @Scripts.Render("~/bundles/jquery")

    <script type="text/javascript">
        $(document).ready(function() {
            // Add click handler
            $("#authenticate-github").click(function () {
                window.open($(this).attr("href"), "github-connect", "width=800, height=600, location=0", true);

                return false;
            });
        });
    </script>
}
``` 

I also need to change the redirect URI which I pass in to the `ChallengeResult`. Currently it redirects back to the `Index` action which means that after the user has been authenticated, the home page will be displayed in the popup window. Instead I want the popup window to close. So I update the `ChallengeResult` to redirect back to a new `AuthorizeGitHubCallback` action what will render a script which will close the popup dialog:

``` csharp
public ActionResult AuthorizeGitHub()
{
    return new ChallengeResult("GitHub", Url.Action("AuthorizeGitHubCallback"));
}

public ActionResult AuthorizeGitHubCallback()
{
    // Render output script to close the popup window
    return Content(@"
        <script>
            window.close();
        </script>
    ");
}
``` 

So now when the user clicks on the button it opens the GitHub authentication in a popup window:

![](/assets/images/popup-oauth-authentication-with-aspnet-and-signalr/popup-oauth-dialog.png)

After the user has authenticated with GitHub the popup window will automatically close, but the user still sees the old home page with the "Authenticate with GitHub" button. They have to manually refresh the browser window to display the user information which was retrieved from GitHub. 

To refresh the home page automatically we will implement a bit of SignalR magic...

## Automatically refresh the home page with SignalR

First install SignalR via Nuget:

``` text
Install-Package Microsoft.AspNet.SignalR
```

Next make sure you map the SignalR hubs to the OWIN pipeline by calling `app.MapSignalR()` in your `Startup` class.

``` csharp
public class Startup
{
    public void Configuration(IAppBuilder app)
    {
	    ...

        app.MapSignalR();
	}
 }
``` 

Next we will need to create a SignalR hub. We can just create an empty class as there is no need for any server side methods. We will only define one method to refresh the page and that will be on the client-side. 

``` csharp
public class AuthenticationHub : Hub
{
     
}
```

Next we need to update the client side script to initialise the SignalR connection and hook a few other things up:

``` html
<script src="~/Scripts/jquery.signalR-2.2.0.min.js"></script>
<script src="~/signalr/hubs"></script>

<script type="text/javascript">
    $(document).ready(function() {
        // Reference the auto-generated proxy for the hub.  
        var authenticationHub = $.connection.authenticationHub;

        // Create a function that the hub can call back to refresh the page
        authenticationHub.client.refreshPage = function () {
            // refresh the page over here..
        };

        // Start the connection
        $.connection.hub.start();

        // Add click handler
        $("#authenticate-github").click(function () {
            window.open($(this).attr("href") + "?connectionId=" + $.connection.hub.id, "github-connect", "width=800, height=600, location=0", true);

            return false;
        });
    });
</script>
```

The code above does the following:

1. Add the SignalR scripts
2. Reference the proxy for the Authentication Hub we created
3. Create a function to refresh the page, which can be called from the server after the user has authenticated with GitHub. (We will come back and write the code for this shortly.)
4. Start the SignalR connection

You will also notice that I have updated the URL which is passed to the `window.open()` method to append the SignalR connection ID. This is so that when we do the callback from the server that we only request this one client to refresh the page, and not all of the clients.

This of course means that we will also need to change the code for the `AuthorizeGitHub` action to take the connection ID as a parameter and pass it along to the `AuthorizeGitHubCallback` action in the redirect URI for the `ChallengeResult`.

``` csharp
public ActionResult AuthorizeGitHub(string connectionId)
{
    return new ChallengeResult("GitHub", Url.Action("AuthorizeGitHubCallback", new { connectionId = connectionId }));
}
```

And we need to change `AuthorizeGitHubCallback` to take the connection ID as a parameter and call the `refreshPage` method on the client with that specific connection ID:

``` csharp
public ActionResult AuthorizeGitHubCallback(string connectionId)
{
    var hubContext = GlobalHost.ConnectionManager.GetHubContext<AuthenticationHub>();
    hubContext.Clients.Clients(new[] { connectionId }).refreshPage();

    // Render output script to close the popup window
    return Content(@"
        <script>
            window.close();
        </script>
    ");
}
``` 

The last bit is to actually refresh the page. In this example I could just as well refresh the entire page, but in most cases you may only want to refresh a small part of the page, so I will rather demonstrate how to refresh just a portion of the page.

So what I am going to do is to create a new MVC action that will render the `_UserProfilePartial.cshtml` and return it to the browser, and replace the content of that area of the page with the newly rendered partial.

```csharp
public async Task<ActionResult> UserProfilePartial()
{
    var userProfile = await GetUserProfile();

    return PartialView("_UserProfilePartial", userProfile);
}
```

And finally implement the refresh page method in my client side script to do an AJAX call to `/Home/UserProfilePartial` path and replace the content of the `div` with the ID "user-profile" with the content returned by the AJAX call. 

So here is what my final `index.cshtml` looks like:

``` html
@model OAuthPopupAuthentication.Models.UserDisplayModel
@{
    ViewBag.Title = "Home Page";
}

<div id="user-profile">
    @Html.Partial("_UserProfilePartial", Model)
</div>

@section Scripts {

    @Scripts.Render("~/bundles/jquery")
    
    <script src="~/Scripts/jquery.signalR-2.2.0.min.js"></script>
    <script src="~/signalr/hubs"></script>

    <script type="text/javascript">
        $(document).ready(function() {
            // Reference the auto-generated proxy for the hub.  
            var authenticationHub = $.connection.authenticationHub;

            // Create a function that the hub can call back to refresh the page
            authenticationHub.client.refreshPage = function () {
                $.get("/Home/UserProfilePartial", function (data) {
                    $("#user-profile").html(data);
                });
            };

            // Start the connection
            $.connection.hub.start();

            // Add click handler
            $("#authenticate-github").click(function () {
                window.open($(this).attr("href") + "?connectionId=" + $.connection.hub.id, "github-connect", "width=800, height=600, location=0", true);

                return false;
            });
        });
    </script>
}
```

## Testing it out

So let's test it all out. When I run the application I get a window which prompts me to authenticate:

![](/assets/images/popup-oauth-authentication-with-aspnet-and-signalr/not-authenticated.png)

Clicking on the button, I will get the GitHub authentication dialog in a popup window:

![](/assets/images/popup-oauth-authentication-with-aspnet-and-signalr/popup-oauth-dialog.png)

And finally after authenticating in GitHub the popup window will automatically close, and the section of my home page which displays the user profile will be automatically updated:

![](/assets/images/popup-oauth-authentication-with-aspnet-and-signalr/authenticated.png) 

## Conclusion

In this blog post demonstrated how you can use ASP.NET, OAuth, SignalR and a bit of jQuery together to create a simple popup authentication for an ASP.NET website.

If you are new to SignalR, please review the [Getting Started with SignalR 2 and MVC 5](http://www.asp.net/signalr/overview/getting-started/tutorial-getting-started-with-signalr-and-mvc) tutorial on the ASP.NET website.
