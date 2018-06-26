---
date: 2016-07-12T00:00:00Z
description: |
  The JSON Web Token middleware in ASP.NET Core knows how to interpret roles inside your JWT.
tags:
- asp.net
- asp.net core
- JWT
- JSON Web Tokens
- authentication
title: Using Roles with the ASP.NET Core JWT middleware
url: /blog/using-roles-with-the-jwt-middleware/
---

Here is a great find: The JWT middleware in ASP.NET Core knows how to interpret a "roles" claim inside your JWT payload, and will add the appropriate claims to the `ClaimsIdentity`. This makes using the `[Authorize]` attribute with Roles very easy.

This is best demonstrated with a simple example.

First of all I head over to [JWT.io](https://www.jwt.io) and create a JSON Web Token with the following payload:

```json
{
  "iss": "http://www.jerriepelser.com",
  "aud": "blog-readers",
  "sub": "123456",
  "exp": 1499863217,
  "roles": ["Admin", "SuperUser"]
}
```

Note the array of roles in the "roles" claim.

This is an HS256 token and signed with the secret "mysuperdupersecret", as can be seen in the following screenshot:

![](/assets/images/2016-07-12-using-roles-with-the-jwt-middleware/jwt_io.png)

In my ASP.NET Core application I am configuring the JWT middleware:

```csharp
public class Startup
{
    public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
    {
        var keyAsBytes = Encoding.ASCII.GetBytes("mysuperdupersecret");

        var options = new JwtBearerOptions
        {
            TokenValidationParameters =
            {
                ValidIssuer = "http://www.jerriepelser.com",
                ValidAudience = "blog-readers",
                IssuerSigningKey = new SymmetricSecurityKey(keyAsBytes)
            }
        };
        app.UseJwtBearerAuthentication(options);

        app.UseMvc();
    }
}
```

When I make a request to my API with the JWT created above, the array of roles in the "roles" claim in the JWT will automatically be added as claims with the type **http://schemas.microsoft.com/ws/2008/06/identity/claims/role** to my `ClaimsIdentity`.

You can test this by creating the following simple API method that returns the user's claims:

```csharp
public class ValuesController : Controller
{
    [Authorize]
    [HttpGet("claims")]
    public object Claims()
    {
        return User.Claims.Select(c =>
        new
        {
            Type = c.Type,
            Value = c.Value
        });
    }
}
```

So when I make a call to the `/claims` endpoint above, and pass the JWT generated before, I will get the following JSON returned:

```json
[
  {
    "type": "iss",
    "value": "http://www.jerriepelser.com"
  },
  {
    "type": "aud",
    "value": "blog-readers"
  },
  {
    "type": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier",
    "value": "123456"
  },
  {
    "type": "exp",
    "value": "1499863217"
  },
  {
    "type": "http://schemas.microsoft.com/ws/2008/06/identity/claims/role",
    "value": "Admin"
  },
  {
    "type": "http://schemas.microsoft.com/ws/2008/06/identity/claims/role",
    "value": "SuperUser"
  }
]
```

Where this gets really interesting is when you consider that passing `Roles` to the `[Authorize]` will actually look whether there is a claim of type **http://schemas.microsoft.com/ws/2008/06/identity/claims/role** with the value of the role(s) you are authorizing.

This means that I can simply add `[Authorize(Roles = "Admin")]` to any API method, and that will ensure that only JWTs where the payload contains the claim "roles" containing the value of **Admin** in the array of roles will be authorized for that API method. 

```csharp
public class ValuesController : Controller
{
    [Authorize(Roles = "Admin")]
    [HttpGet("ping/admin")]
    public string PingAdmin()
    {
        return "Pong";
    }
}
```

This makes things very easy.

What makes this doubly interesting is that this works with the OpenID Connect middleware as well. So in other words, if the ID Token returned when you authorize a user using the OIDC middleware contains a "roles" claim, the exact samle principle applies - simply decorate the MVC controllers with `[Authorize(Roles = "Admin")]` and only users whose ID Token contains those claims will be authorized.

So bottom line: Ensure the "roles" claim of your JWT contains an array of roles assigned to the user, and you can use `[Authorize(Roles = "???")]` in your controllers. It all works seamlessly.