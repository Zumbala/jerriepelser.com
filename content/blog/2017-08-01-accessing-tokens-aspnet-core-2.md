---
date: 2017-08-01T00:00:00Z
description: |
  The ASP.NET Core OIDC middleware allow you to save the ID Token and Access Token, so you can access these in your application.
tags:
- .net core
- asp.net
- asp.net core
- auth0
- jwt
- remote map
title: Accessing the OIDC tokens in ASP.NET Core 2.0
url: /blog/accessing-tokens-aspnet-core-2/
---

Earlier the year I wrote a blog post which described how to [access the JWT Bearer token](/blog/aspnetcore-jwt-saving-bearer-token-as-claim/) when using ASP.NET Core 2.0. Though that was specifically for when using the JWT middleware, you could also use that technique when using the OIDC middleware.

## In ASP.NET Core 1.1

So for example, in ASP.NET Core 1.x, if you wanted to access the tokens (`id_token`, `access_token` and `refresh_token`) from your application, you could set the `SaveTokens` property when registering the OIDC middleware:

```csharp
// Inside your Configure method
app.UseOpenIdConnectAuthentication(new OpenIdConnectOptions("Auth0")
{
    // Set all your OIDC options...

    // and then set SaveTokens to save tokens to the AuthenticationProperties
    SaveTokens = true
});
```

You would then subsequently be able to retrieve those tokens by calling `GetAuthenticateInfoAsync` inside your controllers, and using the result to retreive the tokens, for example:

```csharp
// Inside on of your controllers
if (User.Identity.IsAuthenticated)
{
    var authenticateInfo = await HttpContext.Authentication.GetAuthenticateInfoAsync("Auth0");
    string accessToken = authenticateInfo.Properties.Items[".Token.access_token"];
    string idToken = authenticateInfo.Properties.Items[".Token.id_token"];
}
```

## In ASP.NET Core 2.0

In ASP.NET Core 2.0 this has changed. Firstly, you now register your OIDC middleware inside `ConfigureServices` as follows (making sure to set `SaveTokens` to `true`):

```csharp
// Inside your ConfigureServices method
services.AddAuthentication(options => {
    options.DefaultAuthenticateScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultSignInScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie()
.AddOpenIdConnect(options => {
    // Set all your OIDC options...

    // and then set SaveTokens to save tokens to the AuthenticationProperties
    options.SaveTokens = true;
});
```

You would then subsequently be able to retrieve those tokens by calling `GetTokenAsync` for each of the tokens you want to access. The code sample below shows how to access the `access_token` and the `id_token`:

```csharp
// Inside on of your controllers
if (User.Identity.IsAuthenticated)
{
    string accessToken = await HttpContext.GetTokenAsync("access_token");
    string idToken = await HttpContext.GetTokenAsync("id_token");

    // Now you can use them. For more info on when and how to use the 
    // access_token and id_token, see https://auth0.com/docs/tokens
}
```

Actually a lot(!) has changed with regards to authentication, and I would suggest reading [this GitHub issue](https://github.com/aspnet/Security/issues/1310)
