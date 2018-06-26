---
title: "Implementing an OpenIddict Authorization server: Social Login with GitHub"
description: |
  Expanding the OpenIddict authorization server to allow users to sign in with their GitHub accoouts.
date: 2018-03-01
tags:
- asp.net
- asp.net core
- openiddict
- github
- authorization
url: /blog/implementing-openiddict-authorization-server-part-2
---

In the [previous blog post](/blog/implementing-openiddict-authorization-server-part-1) we created a very basic OIDC authorization server with [OpenIddict](https://github.com/openiddict/openiddict-core). We ended off that blog post by returning hard-coded user information from the authorization endpoint. In this blog post, we'll implement proper user authentication by allowing users to sign in with their GitHub account.

Earlier in the year I [wrote a blog post](/blog/authenticate-oauth-aspnet-core-2/) demonstrating how to configure GitHub authentication using the ASP.NET Core generic OAuth 2.0 provider. In this instance I will be using the GitHub provider from the [aspnet-contrib/AspNet.Security.OAuth.Providers
](https://github.com/aspnet-contrib/AspNet.Security.OAuth.Providers) project. This uses the generic OAuth 2.0 provider under the covers but takes away some of the manual labour involved.

## Registering a GitHub application

First, we'll need to register a GitHub application, so head over to your [GitHub developer settings](https://github.com/settings/developers):

![](/assets/images/2018-03-01-implementing-openiddict-authorization-server-part-2/oauth-apps.png)

Click on the button to **Register a new application**, and complete the information for your application. For the **Authorization callback URL**, specify the URL of your ASP.NET Core application with the path `/signin-github`. In my case this is `http://localhost:57336/signin-github`:

![](/assets/images/2018-03-01-implementing-openiddict-authorization-server-part-2/register-oauth-app.png)

Once done you can click the **Register application button**. Take note of the values for **Client ID** and **Client Secret**, as you will need those shortly when registering the GitHub authentication provider.

![](/assets/images/2018-03-01-implementing-openiddict-authorization-server-part-2/oauth-app-settings.png)

## Registering the GitHub authentication provider

To register the GitHub authentication handler, we'll need to install the `AspNet.Security.OAuth.GitHub` NuGet package first:

```text
dotnet add package AspNet.Security.OAuth.GitHub --version 2.0.0-rc2-final
```

Then head over to the `ConfigureServices` method of your `Startup` class and add the GitHub authentication handler to the authentication services:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContext<AuthorizationDbContext>(options =>
    {
        // Code omitted for brevity
    });

    services.AddMvc();

    services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
        .AddCookie()
        .AddGitHub(options =>
        {
            options.ClientId = Configuration["GitHub:ClientId"];
            options.ClientSecret = Configuration["GitHub:ClientSecret"];

            options.Scope.Add("user:email");

            options.Events = new OAuthEvents
            {
                OnCreatingTicket = context =>
                {
                    // Add claim for avatar
                    string avatar = context.User.Value<string>("avatar_url");
                    context.Identity.AddClaim(new Claim("github:avatar", avatar, ClaimValueTypes.String, context.Options.ClaimsIssuer));

                    return Task.CompletedTask;
                }
            };
        })
        .AddOAuthValidation();

    services.AddOpenIddict(options =>
    {
        // Code omitted for brevity
    });
}
```

Note that we add the **user:email** scope to ensure we get the user's email address from GitHub ([see a list of GitHub scopes](https://developer.github.com/apps/building-oauth-apps/scopes-for-oauth-apps/)). 

By default the GitHub authentication handler is [a bit sparse](https://github.com/aspnet-contrib/AspNet.Security.OAuth.Providers/blob/ec82da32baa9a10de49eb5e2060d7167a64174d4/src/AspNet.Security.OAuth.GitHub/GitHubAuthenticationOptions.cs#L29-L33) in terms of the claims it adds to the `ClaimsIdentity`. I want the user's `avatar_url` attribute added as a claim as well, so in order to do that, we need to handle the `OnCreatingTicket` event to extract it from the user payload returned from GitHub.

In the code snippet above, we are reading the values for the **ClientId** and **ClientSecret** properties from the configuration settings, so be sure to update your configuration file with the values returned when you registered the GitHub application:

```json
{
  "GitHub": {
    "ClientId": "b5c5f371b4278c6dd174",
    "ClientSecret": "ee97c3d2862a437aace2a40630b485bbe06bb104" 
  }
}
```

In my case, I stored these values using the [ASP.NET Core Secret Manager](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets?tabs=visual-studio#secret-manager).

## Updating the authorization endpoint

Previously, we returned hard-coded user information from the authorization endpoint. It is now time to authenticate the user with GitHub and then return the authentication user's information from our OIDC authorization server.

The first thing we'll do is to check whether a user is authenticated. If not, we will challenge the GitHub authentication handler which will send the user off to GitHub to authenticate.

```csharp
[HttpGet("~/connect/authorize")]
public async Task<IActionResult> Authorize(OpenIdConnectRequest request)
{
    // Check if user is authenticated
    if (!User.Identity.IsAuthenticated)
        return Challenge("GitHub");

    //...
}
```

Once the user has signed in to GitHub, GitHub will complete the authorization flow, and the ASP.NET Core authentication middleware will store the GitHub user's information in the cookie. At this point, the authorization endpoint will be invoked again, since it originated the challenge of the GitHub middleware. 

This time, however, the user will be authenticated, so we can retrieve the user's information (name, email etc.), and create an `AuthenticationTicket` with those claims. We will use the same values returned from GitHub for the relevant OIDC claims. For example, the user's GitHub user ID will be returned as the value of the `sub` claim, their name on GitHub as the `name` claim, etc.

Finally, same as last time, we return a `SignInResult`:

```csharp
[HttpGet("~/connect/authorize")]
public async Task<IActionResult> Authorize(OpenIdConnectRequest request)
{
    // Check if user is authenticated
    if (!User.Identity.IsAuthenticated)
        return Challenge("GitHub");

    // Create a new ClaimsPrincipal containing the claims that
    // will be used to create an id_token, a token or a code.
    var claims = new List<Claim>();
    claims.Add(new Claim(OpenIdConnectConstants.Claims.Subject, User.FindFirstValue(ClaimTypes.NameIdentifier)));
    claims.Add(new Claim(OpenIdConnectConstants.Claims.Name, User.FindFirstValue(ClaimTypes.Name)));
    claims.Add(new Claim(OpenIdConnectConstants.Claims.Email, User.FindFirstValue(ClaimTypes.Email)));
    claims.Add(new Claim(OpenIdConnectConstants.Claims.EmailVerified, "true")); // We'll assume email is verified since we get it from GitHub
    claims.Add(new Claim(OpenIdConnectConstants.Claims.Picture, User.FindFirstValue("github:avatar")));
    var identity = new ClaimsIdentity(claims, "OpenIddict");
    var principal = new ClaimsPrincipal(identity);

    // Create a new authentication ticket holding the user identity.
    var ticket = new AuthenticationTicket(principal,
        new AuthenticationProperties(), 
        OpenIdConnectServerDefaults.AuthenticationScheme);

    // Returning a SignInResult will ask OpenIddict to issue the appropriate access/identity tokens.
    return SignIn(ticket.Principal, ticket.Properties, ticket.AuthenticationScheme);
}
```

> BTW, if you do not understand the standard ASP.NET Core authentication flow, then I **strongly suggest** you read [ASP.NET Core 2.0 Authentication and Authorization System Demystified](https://digitalmccullough.com/posts/aspnetcore-auth-system-demystified.html)

## Testing it out

With that in place, we can test out our new authentication flow. Head over to the [OpenID Connect Debugger](https://oidcdebugger.com/) again, and invoke an authentication flow:

![](/assets/images/2018-03-01-implementing-openiddict-authorization-server-part-2/oidc-debugger-request.png)

This time the user will be redirected to GitHub where they will need to sign in and requested to authorize our OIDC server to get access to their information:

![](/assets/images/2018-03-01-implementing-openiddict-authorization-server-part-2/github-authorization-dialog.png)

Once the user has authorized access to our OIDC server, they will be returned to the OIDC server which in turn will redirect the user to the OIDC Debugger:

![](/assets/images/2018-03-01-implementing-openiddict-authorization-server-part-2/oidc-debugger-result.png)

## Inspecting the ID Token

The **ID Token** which returned from our authorization server, and displayed in the OIDC Debugger, is a JSON Web Token. So copy value for the ID Token and head over to the [jwt.io](https://jwt.io/) website to inspect the token:

![](/assets/images/2018-03-01-implementing-openiddict-authorization-server-part-2/inspect-id-token.png)

Notice that `name`, `email`, `email_verified` and `picture` claims which we added to the `ClaimsIdentity` returned in the `AuthenticationTicket` is not present in the ID Token. The reason for this is that we need to tell OpenIddict which of the claims on the `ClaimsIdentity` should be returned in the ID Token and Access Token respectively.

## Specifying claims destinations

OpenIddict supplies extension methods on `ClaimsIdentity`, allowing us to pass in the destinations associated with the claim. In other words whether the claim should be added to the ID Token and/or Access Token:

![](/assets/images/2018-03-01-implementing-openiddict-authorization-server-part-2/openiddict-add-claim.png)

So let's update the section of our `Authorize` method to use these extentions methods and specify that the `name`, `email`, `email_verified` and `picture` claims should be added to the ID Token:

```csharp
[HttpGet("~/connect/authorize")]
public async Task<IActionResult> Authorize(OpenIdConnectRequest request)
{
    Debug.Assert(request.IsAuthorizationRequest(),
        "The OpenIddict binder for ASP.NET Core MVC is not registered. " +
        "Make sure services.AddOpenIddict().AddMvcBinders() is correctly called.");

    // Check if a user is authenticated. If not, challenge the GitHub authentication handler
    if (!User.Identity.IsAuthenticated)
        return Challenge("GitHub");

    // Create a new ClaimsPrincipal containing the claims that
    // will be used to create an id_token, a token or a code.
    var identity = new ClaimsIdentity("OpenIddict");
    identity.AddClaim(OpenIdConnectConstants.Claims.Subject, User.FindFirstValue(ClaimTypes.NameIdentifier), 
        OpenIdConnectConstants.Destinations.AccessToken, OpenIdConnectConstants.Destinations.IdentityToken);
    identity.AddClaim(OpenIdConnectConstants.Claims.Name, User.FindFirstValue(ClaimTypes.Name),
        OpenIdConnectConstants.Destinations.IdentityToken);
    identity.AddClaim(OpenIdConnectConstants.Claims.Email, User.FindFirstValue(ClaimTypes.Email),
        OpenIdConnectConstants.Destinations.IdentityToken);
    identity.AddClaim(OpenIdConnectConstants.Claims.EmailVerified, "true",
        OpenIdConnectConstants.Destinations.IdentityToken); // We'll assume email is verified since we get it from GitHub
    identity.AddClaim(OpenIdConnectConstants.Claims.Picture, User.FindFirstValue("github:avatar"),
        OpenIdConnectConstants.Destinations.IdentityToken);
    var principal = new ClaimsPrincipal(identity);

    // Create a new authentication ticket holding the user identity.
    var ticket = new AuthenticationTicket(principal,
        new AuthenticationProperties(), 
        OpenIdConnectServerDefaults.AuthenticationScheme);
            
    // Returning a SignInResult will ask OpenIddict to issue the appropriate access/identity tokens.
    return SignIn(ticket.Principal, ticket.Properties, ticket.AuthenticationScheme);
}
```

Now, go back to the OIDC Debugger and run through the authorization flow again. This time around, when we inspect the ID Token, you will notice that the `name`, `email`, `email_verified` and `picture` claims are present:

![](/assets/images/2018-03-01-implementing-openiddict-authorization-server-part-2/inspect-id-token-2.png)

### But what about scopes?

Typically you will pass a set of scopes to the authorization endpoint to determine which set of claims should be returned. But, as [Kévin Chalet](http://kevinchalet.com/) (the author of OpenIddict) pointed out to me, this is voluntary and there is [no requirement in the OpenID Connect spec to adhere to this](http://openid.net/specs/openid-connect-core-1_0.html#ScopeClaims).

In this series I am writing a single, first-party client application so I decided to keep things simple and hard-code the claims being returned. If you want to adhere to the scopes being passed in, then please look at [this sample code](https://github.com/openiddict/openiddict-compliance/blob/dev/src/OpenIddict.Compliance/Controllers/AuthorizationController.cs#L154) in the OpenIddict Conformance repo to see how to go about doing that.

## Source code

Source code for this series can be found at [https://github.com/jerriepelser-blog/openiddict-social-login](https://github.com/jerriepelser-blog/openiddict-social-login)

In the next blog post we'll create an Angular client which uses our authorization server to authenticate a user.