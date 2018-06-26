---
date: 2016-12-05T00:00:00Z
description: |
  Demonstrates how you can manage the lifetime of the authentication cookie when using the ASP.NET Core OAuth 2.0 authentication providers
tags:
- asp.net core
- oauth
title: Managing Cookie Lifetime with ASP.NET Core OAuth 2.0 providers
url: /blog/managing-session-lifetime-aspnet-core-oauth-providers/
---

I recently received a support request from a customer regarding the session lifetime once a user has signed in using [Auth0](https://auth0.com) as they wanted the users to remain logged in across browser sessions. For our Auth0 integration with ASP.NET Core [we have written no special middleware](https://auth0.com/blog/auth0-support-for-aspnet-core/) and instead rely on the standard OpenID Connect or OAuth2 middleware for authenticating users in MVC applications.

My initial response to the user was to simply configure the cookie middleware and specify an `ExpireTimeSpan`:

```csharp
// This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    // ...

    app.UseCookieAuthentication(new CookieAuthenticationOptions
    {
        AutomaticAuthenticate = true,
        AutomaticChallenge = true,
        ExpireTimeSpan = TimeSpan.FromDays(7)
    });

    // ...
}
```

Turns out that does not do much. Even when specifying that option the cookie still only remains active for the duration for the session: 

![Cookie duration for lifetime of session only](/assets/images/2016-12-05-managing-session-lifetime-aspnet-core-oauth-providers/cookie-session.png)

## Understanding what ASP.NET Core is doing

So I had a look at the [Cookie Middleware documentation](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/cookie) again and at the bottom of the document there is a section about "Persistent cookies and absolute expiry times". It turns out that you have to specify the cookie persistence options when making a call to `HttpContext.Authentication.SignInAsync(...)`.

The only catch with that is that when using only the OAuth 2.0 or OIDC middleware you never actually make the call to `SignInAsync` - instead the middleware does it for you automatically as can been seen in [the source code](https://github.com/aspnet/Security/blob/2d1c56ce5ccfc15c78dd49cee772f6be473f3ee2/src/Microsoft.AspNetCore.Authentication/RemoteAuthenticationHandler.cs#L110) for the `RemoteAuthenticationHandler` class - which is the base class for both the `OAuthHandler` and `OpenIdConnectHandler` classes of the OAuth 2.0 and OIDC middleware respectively.

Here is the relevant section of code of that class:

```csharp
public abstract class RemoteAuthenticationHandler<TOptions> : AuthenticationHandler<TOptions> where TOptions : RemoteAuthenticationOptions
{
    protected virtual async Task<bool> HandleRemoteCallbackAsync()
    {
        // ...

        context.Properties.Items[AuthSchemeKey] = Options.AuthenticationScheme;

        await Options.Events.TicketReceived(context);

        if (context.HandledResponse)
        {
            Logger.SigninHandled();
            return true;
        }
        else if (context.Skipped)
        {
            Logger.SigninSkipped();
            return false;
        }

        await Context.Authentication.SignInAsync(Options.SignInScheme, context.Principal, context.Properties);

        // Default redirect path is the base path
        if (string.IsNullOrEmpty(context.ReturnUri))
        {
            context.ReturnUri = "/";
        }

        Response.Redirect(context.ReturnUri);
        return true;
    }
}
```

So once the authentication with the remote authentication handler has occurred, the user will be signed in with the `SignInScheme` of the relevant `AuthenticationOptions` instance. You usually specify this `SignInScheme` when you configure the authentication services in the `ConfigureServices` method of your `Startup` class, e.g.

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddAuthentication(options =>
    {
        options.SignInScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    });

    // Add framework services.
    services.AddMvc();
}
```

The value of `CookieAuthenticationDefaults.AuthenticationScheme` is **"Cookies"**, which is the same default value which the Cookie Middleware uses when you register it using 

```csharp
app.UseCookieAuthentication(new CookieAuthenticationOptions
{
    AutomaticAuthenticate = true,
    AutomaticChallenge = true
});
```

So that is how the OAuth 2.0 / OIDC middleware signs the user into using the cookie authentication middleware. That means that on every subsequent request the cookie middleware authenticates the user.

## Option 1: Configuring the `AuthenticationProperties` in the `OnTicketReceived` event

When you look at the code of `RemoteAuthenticationHandler` which I linked to above, you will see that a few lines above the call to `SignInAsync` there is a call to `Options.Events.TicketReceived()` which will fire the `OnTicketReceived` event in our middleware. 

Passed as a parameter to that even is an instance of `TicketRecievedContext` which contains a property of type `AuthenticationProperties` called `Properties` which is ultimately passed in the call to the  `SignInAsync` which is what the ASP.NET Core documentation said is where we should configure the cookie persistence options.

So now with all that knowledge, all we really have to do is to handle the `OnTicketReceived` event when registering our OAuth 2.0 (or OIDC) middleware, and set the correct values to make the cookie persistent for 7 days:

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    //...

    app.UseCookieAuthentication(new CookieAuthenticationOptions
    {
        AutomaticAuthenticate = true,
        AutomaticChallenge = true
    });

    app.UseGitHubAuthentication(new GitHubAuthenticationOptions
    {
        ClientId = "...",
        ClientSecret = "...",
        Scope = { "user:email" },
        Events = new OAuthEvents()
        {
            OnTicketReceived = context =>
            {
                context.Properties.IsPersistent = true;
                context.Properties.ExpiresUtc = DateTimeOffset.UtcNow.AddDays(7);

                return Task.FromResult(0);
            }
        }
    });

    // ...
}
```

And with that in place, when a user signs in to my application you can see that the cookie is now set to expire in 7 days (this blog post was written on 5 December 2016):

![Cookie expires in 7 days](/assets/images/2016-12-05-managing-session-lifetime-aspnet-core-oauth-providers/cookie-7days.png)

## Option 2: Call `SignInAsync` ourselves in the `OnTicketReceived`

Looking at the source code for `RemoteAuthenticationHandler` again, there is another option which becomes evident. You will notice that there is an `if` statement which checks whether the `HandledResponse` property of the `TicketRecievedContext` was set when the `OnTicketReceived` event was handled. This suggest that we can in fact handle the sign in ourselves and then just indicate that we have done so.

Here is how we would do that:

```csharp
app.UseGitHubAuthentication(new GitHubAuthenticationOptions
{
    ClientId = "...",
    ClientSecret = "...",
    Scope = { "user:email" },
    Events = new OAuthEvents()
    {
        OnTicketReceived = context =>
        {
            // Sign the user in ourselves
            context.HttpContext.Authentication.SignInAsync(context.Options.SignInScheme, context.Principal,
                new AuthenticationProperties
                {
                    IsPersistent = true,
                    ExpiresUtc = DateTimeOffset.UtcNow.AddDays(7)
                });

            // Indicate that we handled the login
            context.HandleResponse();

            // Default redirect path is the base path
            if (string.IsNullOrEmpty(context.ReturnUri))
            {
                context.ReturnUri = "/";
            }

            context.Response.Redirect(context.ReturnUri);

            return Task.FromResult(0);
        }
    }
});
```

Notice in the code above that if you want to go this route you will also be responsible for redirecting the user onwards after you have signed them in. 

## Option 3: Storing remote authentication details in 2nd cookie

There is a 3rd option, and that is the route which ASP.NET Identity takes. It has 2 cookies, namely a "main" cookie which authenticates the user, and a second, intermediate cookie in which the user's information is stored when they sign in using an external login provider such as any of the OAuth 2.0 or OIDC proviers.

Here is the basic priniciples of how this approach will work:

1. 2 sets of cookie middleware is registered
    * One which is the "main" cookie (let's call this one the **Application Cookie**) and which is the one that authenticates the user (`AutomaticAuthenticate` and `AutomaticChallenge` is set to true)
    * A second, temporary cookie (we'll call this one the **Remote Authentication Cookie**) in which the login information received from the OAuth 2.0 provider will be stored  
2. The "default" `SignInScheme` for authentication will be set to the **Remote Authentication Cookie**. This means that the OAuth 2.0 middleware will sign in to the **Remote Authentication Cookie** and NOT the **Application Cookie**
3. When the OAuth 2.0 middleware is challenged, we'll instruct it to redirect to a new `RemoteLoginCallback` action after the user has authenticated with the OAuth 2.0 provider.
4. This `RemoteLoginCallback` action will retrieve the user's information from the **Remote Authentication Cookie**, and if it is successful it will manually sign the user in to the  **Application Cookie** by making a call to `HttpContext.Authentication.SignInAsync`.

So in this version, this is my abbreviated `Startup` class:

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddAuthentication(options =>
        {
            options.SignInScheme = "RemoteAuthCookie";
        });

        // Add framework services.
        services.AddMvc();
    }

    // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
    public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
    {
        //...
            
        app.UseCookieAuthentication(new CookieAuthenticationOptions
        {
            AuthenticationScheme = "RemoteAuthCookie",
            CookieName = "RemoteAuthCookie",
            AutomaticAuthenticate = false
        });

        app.UseCookieAuthentication(new CookieAuthenticationOptions
        {
            AuthenticationScheme = "ApplicationCookie",
            CookieName = "ApplicationCookie",
            AutomaticAuthenticate = true,
            AutomaticChallenge = true
        });

        app.UseGitHubAuthentication(new GitHubAuthenticationOptions
        {
            ClientId = "e31c9807a8ce042118d2",
            ClientSecret = "d8b883e189c7b6a1874e14f6d73c11d606b53d24",
            Scope = { "user:email" }
        });

        app.UseMvc(routes =>
        {
            routes.MapRoute(
                name: "default",
                template: "{controller=Home}/{action=Index}/{id?}");
        });
    }
}
``` 

And this is the `AccountController` class which handles logging the user in by challenging the OAuth 2.0 middleware, logging the user out, and also handling the callback from the OAuth 2.0 middleware which signs the user in to the main  **Application Cookie**.

```csharp
public class AccountController : Controller
{
    public async Task<IActionResult> Login()
    {
        // Construct the redirect url to go to the RemoteLoginCallback action
        var redirectUrl = Url.Action("RemoteLoginCallback", "Account", new { ReturnUrl = "/" });

        // Ensure we are signed out of the remote cookie auth
        await HttpContext.Authentication.SignOutAsync("RemoteAuthCookie");

        // Challenge the GH provider
        return new ChallengeResult("GitHub", new AuthenticationProperties() { RedirectUri = redirectUrl });
    }

    public IActionResult Logout()
    {
        HttpContext.Authentication.SignOutAsync("GitHub");
        HttpContext.Authentication.SignOutAsync("ApplicationCookie");

        return RedirectToAction("Index", "Home");
    }

    public async Task<IActionResult> RemoteLoginCallback(string returnUrl)
    {
        var auth = new AuthenticateContext("RemoteAuthCookie");

        // Get auth ticket from remote cookie
        await HttpContext.Authentication.AuthenticateAsync(auth);

        if (auth.Accepted)
        {
            // Sign out of remote cookie once we used it
            await HttpContext.Authentication.SignOutAsync("RemoteAuthCookie");

            // Sign the user in
            await HttpContext.Authentication.SignInAsync("ApplicationCookie", auth.Principal, new AuthenticationProperties
            {
                IsPersistent = true,
                ExpiresUtc = DateTimeOffset.UtcNow.AddDays(7)
            });

            return RedirectToLocal(returnUrl);
        }
        else
        {
            // If we don't have an external auth cookie, redirect to login action
            return RedirectToAction(nameof(Login));
        }

    }

    private IActionResult RedirectToLocal(string returnUrl)
    {
        if (Url.IsLocalUrl(returnUrl))
        {
            return Redirect(returnUrl);
        }
        else
        {
            return RedirectToAction(nameof(HomeController.Index), "Home");
        }
    }
}
```

Please note that the official ASP.NET Identity code does some extra checks for XSRF, so do not assume my code is "production ready". If you want to go with this approach please reference the ASP.NET Identity source code, and especially the code for the [GetExternalLoginInfoAsync](https://github.com/aspnet/Identity/blob/bb60afba28d7775e610afafd85cce810c9200374/src/Microsoft.AspNetCore.Identity/SignInManager.cs#L485-L517) method which read the user's information from the external cookie.

## Source code

You can find full source code demonstrating all 3 different approaches at [https://github.com/jerriepelser-blog/oauth-authentication-cookies](https://github.com/jerriepelser-blog/oauth-authentication-cookies)