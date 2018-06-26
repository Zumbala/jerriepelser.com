---
date: 2016-07-07T00:00:00Z
description: |
  I show how you can add extra parameters to the Authorization URL when using the ASP.NET Core OpenID Connect middleware.
tags:
- asp.net
- asp.net core
- openid connect
- oidc
- authentication
title: Adding parameters to the OpenID Connect Authorization URL
url: /blog/adding-parameters-to-openid-connect-authorization-url/
---

I am busy working on some more samples for ASP.NET Core to demonstrate various techniques people can use [Auth0](https://auth0.com/) to authenticate their users. In most of our samples we use the standard OpenID Connect middleware, and one of the things I wanted to do was to pass extra parameters when the request is made to the Authorization endpoint.

At Auth0 we allow users to authenticate with [multiple social and Enterprise providers](https://auth0.com/docs/identityproviders). Usually when the Authorization endpoint is called, we will display [Lock](https://auth0.com/lock) which will promt the user for their username and password, and also allow them to sign in with any of the connected social or enterprise providers.

![](/assets/images/2016-07-07-adding-parameters-to-openid-connect-authorization-url/lock.png) 

We can however also directly invoke any of the social connections, bypassing Lock completely and directing the user directly to the Authorization page for the relevant service. So as an example we can send the user directly to the Google login by passing along the query string parameter `connection=google-oauth2`.

So how do you do this when using the OpenID Connect middleware?

All you need to do is handle the `OnRedirectToIdentityProvider` event when configuring the `OpenIdConnectOptions`, and add the exta query string parameters by calling the `ProtocolMessage.SetParameter` method on the supplied `RedirectContext`

```csharp
app.UseOpenIdConnectAuthentication(new OpenIdConnectOptions("Auth0")
{
    // Set the authority to your Auth0 domain
    Authority = "https://YOUR_AUTH0_DOMAIN",

    // Configure the Auth0 Client ID and Client Secret
    ClientId = "CLIENT ID",
    ClientSecret = "CLIENT SECRET",

    // Do not automatically authenticate and challenge
    AutomaticAuthenticate = false,
    AutomaticChallenge = false,

    // Set response type to code
    ResponseType = "code",

    // Set the callback path 
    CallbackPath = new PathString("/signin-auth0"),

    // Configure the Claims Issuer to be Auth0
    ClaimsIssuer = "Auth0",

    Events = new OpenIdConnectEvents
    {
        OnRedirectToIdentityProvider = context =>
        {
            context.ProtocolMessage.SetParameter("connection", "google-oauth2");

            return Task.FromResult(0);
        }
    }
});
```

Now the user will be sent directly to the Google login page whenever the OIDC middleware is invoked.

This however means that the user will always be directed to sign in with their Google account. What if we want to make this configurable somehow?

At the moment the `Login` action in the `AccountController` which issues the challenge to the OIDC middleware looks as follows:

```csharp
public IActionResult Login()
{
    return new ChallengeResult("Auth0", new AuthenticationProperties() { RedirectUri = "/" });
}
```

What we need to do is add a `connection` parameter to the `Login` action and then if the user passed in a value for that parameter we can add it to the `Items` dictionary of the `AuthenticationProperties` instance which is passed along with the challenge:   

```csharp
public IActionResult Login(string connection)
{
    var properties = new AuthenticationProperties() {RedirectUri = "/"};

    if (!string.IsNullOrEmpty(connection))
        properties.Items.Add("connection", connection);

    return new ChallengeResult("Auth0", properties);
}
```

And then also change the `OnRedirectToIdentityProvider` delegate to check if the connection property was passed along, and if it was, append the value to the `ProtocolMessage` parameters:

```csharp
app.UseOpenIdConnectAuthentication(new OpenIdConnectOptions("Auth0")
{
    // Set the authority to your Auth0 domain
    Authority = "https://YOUR_AUTH0_DOMAIN",

    // Configure the Auth0 Client ID and Client Secret
    ClientId = "CLIENT ID",
    ClientSecret = "CLIENT SECRET",

    // Do not automatically authenticate and challenge
    AutomaticAuthenticate = false,
    AutomaticChallenge = false,

    // Set response type to code
    ResponseType = "code",

    // Set the callback path 
    CallbackPath = new PathString("/signin-auth0"),

    // Configure the Claims Issuer to be Auth0
    ClaimsIssuer = "Auth0",

    Events = new OpenIdConnectEvents
    {
        OnRedirectToIdentityProvider = context =>
        {
            if (context.Properties.Items.ContainsKey("connection"))
                context.ProtocolMessage.SetParameter("connection", context.Properties.Items["connection"]);

            return Task.FromResult(0);
        }
    }
});
```

Now, when you go to `http://YOUR_URL/Account/Login`, the OIDC middleware will get invoked and Auth0 Lock will be display as always. However if you go to `http://YOUR_URL/Account/Login?connection=google-oauth2` then the user will be sent directly to the Google authorization page. Likewise, if you go to `http://YOUR_URL/Account/Login?connection=github`, the user will be sent directly to the GitHub authorization page. 