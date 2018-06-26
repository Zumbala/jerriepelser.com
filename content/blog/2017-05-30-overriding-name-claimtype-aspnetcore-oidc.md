---
date: 2017-05-30T00:00:00Z
description: |
  Overriding the NameClaimType when in the ASP.NET Core OpenID Connect middleware allow you to specify a different claim from the ID Token to use as the name of the subject.
tags:
- .net core
- asp.net
- asp.net core
- oidc
title: Overriding the NameClaimType when using the ASP.NET Core OpenID Connect middleware
url: /blog/overriding-name-claimtype-aspnetcore-oidc/
---

When using the ASP.NET Core OpenID Connect middleware, after a user has signed in you can access the name of the user by using the `User.Identity.Name` property. What this does under the cover is to look for the `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name` claim type and then return that value.

In the case of Auth0, we actually return a user's name in the `name` claim of an `id_token` and this does not get automatically mapped to the `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name` claim type. So previously in our [ASP.NET Core Quickstart](https://auth0.com/docs/quickstart/webapp/aspnet-core) I actually wrote a whole bunch of code demonstrating how to handle the `OnTicketReceived` event to read the `name` claim, and use that value to add a claim of the type `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name` to ensure that everything works correctly.

Turns out there is a much easier way [as a user pointed out to me](https://github.com/auth0-samples/auth0-aspnetcore-mvc-samples/issues/23).

All you need to do is to specify the `NameClaimType` of the `TokenValidationParameters` property of the `OpenIdConnectOptions`. So in the case of an Auth0 `id_token`, you need to specify the `NameClaimType` with a value of `name`:

```csharp
var options = new OpenIdConnectOptions("Auth0")
{
    // .. some code omitted for brevity ..

    // Set the correct name claim type
    TokenValidationParameters = new TokenValidationParameters
    {
        NameClaimType = "name"
    }
};
app.UseOpenIdConnectAuthentication(options);
```

Now, every time you get the user's name through the `User.Identity.Name` property, it will look for the value of the `name` claim on the user, and return the correct value for the user's name.
