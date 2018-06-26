---
date: 2017-09-12T00:00:00Z
description: |
  Demonstrates how you can configure the Google authentication handler to only allow users from a certain domain to sign in to your application.
tags:
- asp.net core
- google
- oauth
title: Forcing user to sign in with their Google Organization (G Suite) account
url: /blog/forcing-users-sign-in-gsuite-domain-account/
---

Microsoft has a nice document explaining how to allow users to [sign in to your application with their Google accounts](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/social/google-logins?tabs=aspnetcore2x). I was curious to see how one could go about forcing users to sign in with their organizational account. 

Why would you want to do this?

Well, let say your company is using G Suite as its directory service. Only allowing users to sign in with their organizational (G Suite) email address means that you only have to procure users on the G Suite directory service. You don't have to worry about maintaining a separate user store to manage your users, or worry that people from outside of your organization can sign in.

In my case I have a `jerriepelser.com` domain which is managed by G Suite. In this example I will show you how to configure my application to only allow users with a `jerriepelser.com` email address to sign into the application.

We're going to start of with a new MVC project, so go ahead and create the new application using the .NET Core command line tool:

```
dotnet new mvc
```

## Enabling Google Sign In

The first thing you will need to do is to create an app in the Google API Console. You can refer to the [Microsoft documentation](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/social/google-logins?tabs=aspnetcore2x) on how to do that.

Once you have registered the app in Google API Console, add your Client ID and Client Secret to your `appsettings.json` file:

```json
{
  "Authentication": {
    "Google": {
      "ClientId": "...",
      "ClientSecret": "..."
    }
  }
}
```

Register the authentication service in the `ConfigureServices` method of your `Startup` class:

```csharp
// Startup.cs

public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc();

    services.AddAuthentication(options =>
        {
            options.DefaultAuthenticateScheme = CookieAuthenticationDefaults.AuthenticationScheme;
            options.DefaultSignInScheme = CookieAuthenticationDefaults.AuthenticationScheme;
        })
        .AddCookie()
        .AddGoogle("Google", options =>
        {
            options.ClientId = Configuration["Authentication:Google:ClientId"];
            options.ClientSecret = Configuration["Authentication:Google:ClientSecret"];
        });
}
```

Note that I have set the `DefaultAuthenticateScheme` and `DefaultSignInScheme` to cookies. This means that all request will be authenticated using cookies, and once an external provider (such as Google in this case) has authenticated a user, the user will be signed in to the app using a cookie.

In the call to `AddGoogle` I specified the `authenticationScheme` as **Google**. If you don't specify an `authenticationScheme` parameter, it actually defaults to **Google** as well, but I want to be explicit about it so you can understand how the Google authentication provider gets challenged (or triggered) in the `Login` action below.

Also, remember to add the authentication middleware by calling `UseAuthentication` in your `Configure` method:

```csharp
// Startup.cs

public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }
    else
    {
        app.UseExceptionHandler("/Home/Error");
    }

    app.UseStaticFiles();

    app.UseAuthentication();
    
    app.UseMvc(routes =>
    {
        routes.MapRoute(
            name: "default",
            template: "{controller=Home}/{action=Index}/{id?}");
    });
}
```

Next, I create an `AccountController` class with `Login`, `Logout` and `Profile` actions.

```csharp
public class AccountController : Controller
{
    public async Task Login(string returnUrl = "/")
    {
        await HttpContext.ChallengeAsync("Google", new AuthenticationProperties() { RedirectUri = returnUrl } );
    }

    [Authorize]
    public async Task Logout()
    {
        await HttpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme, new AuthenticationProperties
        {
            RedirectUri = Url.Action("Index", "Home")
        });
    }

    [Authorize]
    public IActionResult Profile()
    {
        return View();
    }
}
```

The `Login` action challenges the authentication provider with the scheme of **Google**. So it will invoke the Google authentication handler we specified earlier, because we registered it with the `authenticationSceheme` of **Google**. After successful login it will redirect the user to the value passed in the `returnUrl` parameter, which defaults to the home page.

The `Logout` action will simply sign the user out of the cookie handler by clearing the cookie. After logout it will redirect the user to the home page.

The `Profile` action will display a view with just the list of claims for the user. Nothing fancy, but it allows us to see which claims are being returned by Google.

In the `/Views/Account` folder, create a new view file called `Profile.cshtml`. Here is the code for the view:

```html
@{
    ViewData["Title"] = "Profile";
}
<div class="row">
    <div class="col-md-12">

        <h3>Claims associated with current User</h3>

        <table class="table">
            <thead>
            <tr>
                <th>
                    Claim
                </th>
                <th>
                    Value
                </th>
            </tr>
            </thead>
            <tbody>
            @foreach (var claim in User.Claims)
            {
                <tr>
                    <td>@claim.Type</td>
                    <td>@claim.Value</td>
                </tr>
            }
            </tbody>
        </table>
    </div>
</div>
```

Finally we need to add login and logout buttons to the menu. Head over to the `/Views/Shared/_layout.cshtml` file and in the navbar section, add some links to the `Login` and `Logout` actions. In the case where a user is authenticated, we will also display their name, which links to the `Profile` action.

```html
<!-- some code omitted -->
<nav class="navbar navbar-inverse navbar-fixed-top">
    <div class="container">
        <div class="navbar-header">
            <button type="button" class="navbar-toggle" data-toggle="collapse" data-target=".navbar-collapse">
                <span class="sr-only">Toggle navigation</span>
                <span class="icon-bar"></span>
                <span class="icon-bar"></span>
                <span class="icon-bar"></span>
            </button>
            <a asp-area="" asp-controller="Home" asp-action="Index" class="navbar-brand">GoogleAuth</a>
        </div>
        <div class="navbar-collapse collapse">
            <ul class="nav navbar-nav">
                <li><a asp-area="" asp-controller="Home" asp-action="Index">Home</a></li>
                <li><a asp-area="" asp-controller="Home" asp-action="About">About</a></li>
                <li><a asp-area="" asp-controller="Home" asp-action="Contact">Contact</a></li>
            </ul>
            <ul class="nav navbar-nav navbar-right">
                @if (User.Identity.IsAuthenticated)
                {
                    <li><a asp-controller="Account" asp-action="Profile">Hello @User.Identity.Name!</a></li>
                    <li><a  asp-controller="Account" asp-action="Logout">Logout</a></li>
                }
                else
                {
                    <li><a asp-controller="Account" asp-action="Login">Login</a></li>
                }
            </ul>
        </div>
    </div>
</nav>
<!-- some code omitted -->
```

## Testing it out

Great, so lets test this out by running the application. Click on the **Login** link in the top right corner:

![](/assets/images/2017-09-12-forcing-users-sign-in-gsuite-domain-account/login-link.png)

This redirects me to Google to authenticate. You can see that I am signed in to Google with both my `jerriepelser.com` and my Auth0 email accounts. So let's click on the `jerriepelser.com` account:

![](/assets/images/2017-09-12-forcing-users-sign-in-gsuite-domain-account/google-account-chooser.png)

This redirects me back to my application, and you can see that I am signed in:

![](/assets/images/2017-09-12-forcing-users-sign-in-gsuite-domain-account/signed-in.png)

If I click on the **Hello Jerrie Pelser!** text I will be taken to the Profile page where you can see all the claims that were returned by Google.

![](/assets/images/2017-09-12-forcing-users-sign-in-gsuite-domain-account/profile.png)

## Limiting users to your Organizational domain

Right now I can also actually sign in with my Auth0 Google Account, or any other Google account that I have. We want to limit it only to users on the `jerriepelser.com` domain, but before we do that let's quickly understand what the Google authentication handler is doing.

Once a user has authenticated with Google, the Google authentication handler is going to call the user information endpoint at `https://www.googleapis.com/plus/v1/people/me`. 

That endpoint is going to return a JSON object similar to the following:

```json
{
 "kind": "plus#person",
 "etag": "\"Sh4n9u6EtD24TM0RmWv7jTXojqc/LyG9i73o_qqjFbpRIt1nc9uVTvk\"",
 "emails": [
  {
   "value": "...@jerriepelser.com",
   "type": "account"
  }
 ],
 "objectType": "person",
 "id": "1081425....",
 "displayName": "Jerrie Pelser",
 "name": {
  "familyName": "Pelser",
  "givenName": "Jerrie"
 },
 "image": {
  "url": "https://lh5.googleusercontent.com/-UEA_Kl3MzOE/AAAAAAAAAAI/AAAAAAAAAMQ/WDoAqIJGYTU/photo.jpg?sz=50",
  "isDefault": false
 },
 "isPlusUser": false,
 "language": "en",
 "ageRange": {
  "min": 21
 },
 "verified": false,
 "domain": "jerriepelser.com"
}
```

Next, the Google authentication handler is going to invoke the `OnCreatingTicket` event, passing along an instance of `OAuthCreatingTicketContext`. This context will contain a `User` property which allows us to access the JSON payload returned from Google, which I showed the example of above.

If you are interested in the code that does all of this, you can check it out [over here](https://github.com/aspnet/Security/blob/a53bf093a7d86b35e019c80515c92d7626982325/src/Microsoft.AspNetCore.Authentication.Google/GoogleHandler.cs#L31-L46)

So now let's handle that `OnCreatingTicket` event.

Head back to the `ConfigureServices` method, and let's alter the registration of the Google authentication handler by adding an `OnCreatingTicket` event handler. Inside this event handler we need to inspect the **domain** property of the `User`, to see whether it is in fact **jerriepelser.com**. If it is, then we're happy. If it isn't we throw an exception:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc();

    services.AddAuthentication(options =>
        {
            options.DefaultAuthenticateScheme = CookieAuthenticationDefaults.AuthenticationScheme;
            options.DefaultSignInScheme = CookieAuthenticationDefaults.AuthenticationScheme;
        })
        .AddCookie()
        .AddGoogle("Google", options =>
        {
            options.ClientId = Configuration["Authentication:Google:ClientId"];
            options.ClientSecret = Configuration["Authentication:Google:ClientSecret"];
            options.Events = new OAuthEvents
            {
                OnCreatingTicket = context =>
                {
                    string domain = context.User.Value<string>("domain");
                    if (domain != "jerriepelser.com")
                        throw new GoogleAuthenticationException("You must sign in with a jerriepelser.com email address");

                    return Task.CompletedTask;
                }
            };
        });
}
```

That `GoogleAuthenticationException` class is just a class that inherits from `Exception`. Nothing special. The reason why I am using a custom exception class however, is so that I can display a nicer error message to the user. 

Our application template that was created has an error handler which can be found in the `HomeController` class. This simply displays a generic error message. 

However, in a case like this I want to display the actual error message ("You must sign in with a jerriepelser.com email address") to the user. So let's first alter the `ErrorViewModel` class (located in `/ViewModels/ErrorViewModel.cs`) to also add an `ErrorMessage` property.

```csharp
public class ErrorViewModel
{
    public string ErrorMessage { get; set; }

    public string RequestId { get; set; }

    public bool ShowRequestId => !string.IsNullOrEmpty(RequestId);
}
```

Then we'll alter the `Error` action to retrieve the exeption, and try and cast it to `GoogleAuthenticationException`. If it is a `GoogleAuthenticationException` we'll add the message to the `ErrorMessage` property of the `ErrorViewModel`. Alternatively the `ErrorMessage` property will be `null`.

```csharp
public class HomeController : Controller
{
    // some code omitted for brevity...

    public IActionResult Error()
    {
        var exceptionHandlerFeature = HttpContext.Features.Get<IExceptionHandlerFeature>();
        var exception = exceptionHandlerFeature?.Error as GoogleAuthenticationException;

        return View(new ErrorViewModel
        {
            RequestId = Activity.Current?.Id ?? HttpContext.TraceIdentifier,
            ErrorMessage = exception?.Message
        });
    }
}
```

Let's also alter the `/Views/Shared/Error.cshtml` to display the `ErrorMessage` if there is one. If not, it will display the generic "An error has occurred..." message:

```html
@model ErrorViewModel
@{
    ViewData["Title"] = "Error";
}

<h1 class="text-danger">Error.</h1>

@if (!string.IsNullOrEmpty(Model.ErrorMessage))
{
    <h2 class="text-danger">@Model.ErrorMessage</h2>
}
else
{
    <h2 class="text-danger">An error occurred while processing your request.</h2>
}

@if (Model.ShowRequestId)
{
    <p>
        <strong>Request ID:</strong> <code>@Model.RequestId</code>
    </p>
}

<h3>Development Mode</h3>
<p>
    Swapping to <strong>Development</strong> environment will display more detailed information about the error that occurred.
</p>
<p>
    <strong>Development environment should not be enabled in deployed applications</strong>, as it can result in sensitive information from exceptions being displayed to end users. For local debugging, development environment can be enabled by setting the <strong>ASPNETCORE_ENVIRONMENT</strong> environment variable to <strong>Development</strong>, and restarting the application.
</p>
```

Finally we can give it a spin again. This time if I sign in with my `jerriepelser.com` email, all works as before. But if I try and sign in with my Auth0 account, I get the following error:

![](/assets/images/2017-09-12-forcing-users-sign-in-gsuite-domain-account/domain-error.png)

## Specifying the Hosted Domain parameter

There is one more thing we can do, and that is to specify the **Hosted Domain** or `hd` parameter when the user is redirected to the Google authorization endpoint (see the [docs](https://developers.google.com/identity/protocols/OpenIDConnect#authenticationuriparameters)). Here is the relevant section from those docs:

> By including the domain of the G Suite user (for example, mycollege.edu), you can indicate that the account selection UI should be optimized for accounts at that domain.

So this will basically pre-fill the domain as whatever we pass along (in our case `jerriepelser.com`), and the user will only need to fill in name part of the email address. 

Once again we'll head back to the `ConfigureServices` method, and this time we'll add an `OnRedirectToAuthorizationEndpoint` event. This is simply going to slap that `hd` parameter to the end of the `RedirectUri`, and then redirect the user to that endpoint:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc();

    services.AddAuthentication(options =>
        {
            options.DefaultAuthenticateScheme = CookieAuthenticationDefaults.AuthenticationScheme;
            options.DefaultSignInScheme = CookieAuthenticationDefaults.AuthenticationScheme;
        })
        .AddCookie()
        .AddGoogle("Google", options =>
        {
            options.ClientId = Configuration["Authentication:Google:ClientId"];
            options.ClientSecret = Configuration["Authentication:Google:ClientSecret"];
            options.Events = new OAuthEvents
            {
                OnRedirectToAuthorizationEndpoint = context =>
                {
                    context.Response.Redirect(context.RedirectUri + "&hd=" + System.Net.WebUtility.UrlEncode("jerriepelser.com"));
                    
                    return Task.CompletedTask;
                },
                OnCreatingTicket = context =>
                {
                    string domain = context.User.Value<string>("domain");
                    if (domain != "jerriepelser.com")
                        throw new GoogleAuthenticationException("You must sign in with a jerriepelser.com email address");

                    return Task.CompletedTask;
                }
            };
        });
}
```

Let's try that again.

What happens for me when I do this is that the Google login prompt does not even display. From the app I get redirected to Google, and because I am already signed in with a user with that domain, Google immediately returns that as the authenticated user to your app.

To better understand what is happening, your first need to sign out of your Google account, or open an Incognito browser window:

![](/assets/images/2017-09-12-forcing-users-sign-in-gsuite-domain-account/google-account-chooser-prefilled-domain.png)

Notice in the screenshot above that the domain part is prefilled as `jerriepelser.com`.

You may ask why we still need the check in the `OnCreatingTicket` event. Well the Google docs explicitly states that we **should not rely on this UI optimization to control who can access our app**:

![](/assets/images/2017-09-12-forcing-users-sign-in-gsuite-domain-account/hd-warning.png)

I'm not 100% sure, but I suspect that the `state` parameter being generated by the Google authentication hander will check and prevent someone from messing with the outgoing request. However, just keep that code in place in any case to be safe.

## Code

Source code can be found at [jerriepelser-blog/google-gsuite-oauth](https://github.com/jerriepelser-blog/google-gsuite-oauth).