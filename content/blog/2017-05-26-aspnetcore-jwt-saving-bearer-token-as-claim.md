---
date: 2017-05-26T00:00:00Z
description: |
  When using the ASP.NET Core JWT authentication handler, there are instances in which you may want to access the actual bearer token which was passed to the request.
tags:
- .net core
- asp.net
- asp.net core
- auth0
- jwt
- remote map
title: Access the JWT bearer token when using the JWT middleware in ASP.NET Core
url: /blog/aspnetcore-jwt-saving-bearer-token-as-claim/
---

When using JSON Web Tokens (JWTs) as Bearer tokens in your ASP.NET Core Web API, it may sometimes be required to access the actual token which was passed to the API somewhere else in your API. 

For [Remote Map](http://www.jerriepelser.com/blog/introducing-remote-map/) for example I have the requirement to access the user's full profile under certain conditions. I want to store some of the user's personal information in the local database, and in order to obtain their information I have to call the [/userinfo](https://auth0.com/docs/api/authentication#get-user-info) endpoint of the Auth0 Authentication API.

In order to call the `/userinfo` endpoint, I need to pass the `access_token` along - the same `access_token` which was used as a Bearer Token to call the API endpoint.

In ASP.NET Core there are 2 ways in which you can go about this:

1. Save the token as a claim
2. Save the token in the [AuthenticationProperties](https://docs.microsoft.com/en-us/aspnet/core/api/microsoft.aspnetcore.http.authentication.authenticationproperties)

> **Please note** that the solutions proposed in here is (was) for ASP.NET Core 1.x. In ASP.NET Core 2.0 this can be done much easier as I [described in this blog post](/blog/accessing-tokens-aspnet-core-2/) 

## 1. Save the token as a claim

In order to do this you will need to handle the `OnTokenValidated` event inside the `JwtBearerOptions`. Inside this event you can access the `SecurityToken` property of the `TokenValidatedContext` and cast it to a `JwtSecurityToken`.

Once you have that, you can access the token from `RawData` add it as a claim to the `ClaimsIdentity`

```csharp
var options = new JwtBearerOptions
{
    Audience = "...",
    Authority = "...",
    Events = new JwtBearerEvents
    {
        OnTokenValidated = context =>
        {
            // Add the access_token as a claim, as we may actually need it
            var accessToken = context.SecurityToken as JwtSecurityToken;
            if (accessToken != null)
            {
                ClaimsIdentity identity = context.Ticket.Principal.Identity as ClaimsIdentity;
                if (identity != null)
                {
                    identity.AddClaim(new Claim("access_token", accessToken.RawData));
                }
            }

            return Task.CompletedTask;
        }
    }
};
app.UseJwtBearerAuthentication(options);
```

Now to access if from one of your controller actions, you can simply get the value of the "access_token" claim:

```csharp
string accessToken = User.FindFirst("access_token")?.Value
```

## 2. Save the token in the AuthenticationProperties

The second way to do this is to set the `SaveToken` property of the `JwtBearerOptions` to `true`, which will save the token to the [AuthenticationProperties](https://docs.microsoft.com/en-us/aspnet/core/api/microsoft.aspnetcore.http.authentication.authenticationproperties)

```csharp
var options = new JwtBearerOptions
{
    Audience = "...",
    Authority = "...",
    SaveToken = true
};
app.UseJwtBearerAuthentication(options);
```

Then subsequently if you want to get the value of the `access_token` in one of your controller actions, you will need to retrieve the `AuthenticateInfo`. The `access_token` will be stored inside the `Properties` with the key ".Token.access_token":

```csharp
var authenticateInfo = await HttpContext.Authentication.GetAuthenticateInfoAsync("Bearer");
string accessToken = authenticateInfo.Properties.Items[".Token.access_token"];
```

## Conclusion

In this blog post I demonstrated 2 ways in which you can access the JSON Web Token which you passed as Bearer token to your ASP.NET Core API. I am honestly not sure whether one way is superior to the other, so choose whichever you feel most comfortable with.
