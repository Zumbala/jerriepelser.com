---
date: 2015-04-27T00:00:00Z
description: |
  The ASP.NET OAuth providers do not have to be used in conjunction with ASP.NET Identity. I show you how you can use them on their own.
tags:
- aspnet
- oauth
- authentication
- identity
title: Using the ASP.NET OAuth providers without ASP.NET Identity
url: /blog/using-aspnet-oauth-providers-without-identity/
---

## Introduction

I think very few ASP.NET developers realise that the ASP.NET authentication providers and ASP.NET Identity really have nothing to with each other. It is two completely independent technologies which are brought together nicely by the default ASP.NET MVC project template to allow users to sign in with the social media accounts, or link a social media sign in to their existing user account.

ASP.NET Identity is the technology which provides user management, role management and authentication. It also allow for "external" logins to be linked to a user's account, but those external logins can take any form.

Here is a look at the relevant tables from the ASP.NET Identity database schema:

![](/assets/images/2015-04-27-using-aspnet-oauth-providers-without-identity/identity-table-diagram.png)

In the diagram above, the AspNetUsers table is the central table where the primary user information is stored. Linked to that is the AspNetUserLogins table which allows ASP.NET Identity to store any number of logins associated to that user. An external login contains fields for the LoginProvider and ProviderKey. 

The LoginProvider is the name of the service which provided the login, and the ProviderKey is the unique key associated with the login on that service. So if we take an example where a user is allowed to sign in with GitHub, the LoginProvider will be "GitHub" and the ProviderKey will be the unique identifier of the user on GitHub.

And this is where the ASP.NET external authentication providers come in. What they do is to authenticate a user with some external authentication service and after the user has been authenticated they will return a `ClaimsIdentity`. The `ClaimsIdentity` contains a list of claims (which is basically some properties which are returned from the external service about the user), and one of those claims is the unique identifier of the user on the external service.

So the standard ASP.NET MVC template orchestrates all of this nicely together to make it seamless to you. When a user chooses to sign into your website with their GitHub account, the GitHub OAuth Provider will kick in and authenticate the user on GitHub. After the user is authenticated it returns a `ClaimsIdentity` which contains among other things a claim with the unique ID of the user on GitHub.

The ASP.NET MVC template has code which then extracts that ID, checks whether the user is signed in already and associate and external login with the user, or alternatively it will create a new user and also adds an external login for that user with the `LoginProvider` set to "GitHub" and the `ProviderKey` set to the ID of the user on GitHub. This part uses the ASP.NET Identity APIs.

## An example 

### OWIN Startup class

Let's put together a quick example of authenticating a user with GitHub, extracting the OAuth token returned from GitHub and then displaying the GitHub access token.

Start with creating a new ASP.NET MVC application, and make sure that you set it to use no authentication:

![](/assets/images/2015-04-27-using-aspnet-oauth-providers-without-identity/new-mvc-application.png) 

Next install some Nuget packages:

``` text
install-package Microsoft.Owin.Host.SystemWeb
install-package Microsoft.Owin.Security.Cookies
install-package Owin.Security.Providers
install-package Octokit
```

The first two packages are needed for some OWIN infrastructure, and the third package contains the GitHub authentication provider. The `Octokit` package is a C# wrapper for the GitHub API. 

Next up add an OWIN Startup class to you web project. Select the "Add New Item" from the project menu, and search for the "OWIN Startup Class" template, and name it 'Startup.cs":

![](/assets/images/2015-04-27-using-aspnet-oauth-providers-without-identity/add-owin-startup-class.png) 

Inside the `Configuration` method of your new Startup class you will need to configure Cookie authentication as this is required for the OAuth authentication to function properly, and the OWIN middleware will store the result from the GitHub authentication in this cookie:

``` csharp
app.SetDefaultSignInAsAuthenticationType("ExternalCookie");
app.UseCookieAuthentication(new CookieAuthenticationOptions
{
    AuthenticationType = "ExternalCookie",
    AuthenticationMode = AuthenticationMode.Passive,
    CookieName = ".AspNet.ExternalCookie",
    ExpireTimeSpan = TimeSpan.FromMinutes(5),
});
```  

Register a new application in GitHub and use the Client ID and Client Secret supplied to you to register the GitHub authentication provider:

``` csharp
var options = new GitHubAuthenticationOptions
{
    ClientId = "your client id",
    ClientSecret = "your client secret",
    Provider = new GitHubAuthenticationProvider
    {
        OnAuthenticated = context =>
        {
            context.Identity.AddClaim(new Claim("urn:token:github", context.AccessToken));

            return Task.FromResult(true);
        }
    }
};
app.UseGitHubAuthentication(options);
```

If you do not know how to register an application in GitHub, you can [look at this guide](http://www.oauthforaspnet.com/providers/github/guides/aspnet-mvc5/) I wrote on how to do it.

One important thing to note in the code above is the `OnAuthenticated` handler I wrote. Inside this handler I add the actual access token as a claim to the `ClaimsIdentity`, as that is where I will retrieve it later.

Here is the full code for the Startup class:

``` csharp
public class Startup
{
    public void Configuration(IAppBuilder app)
    {
        app.SetDefaultSignInAsAuthenticationType("ExternalCookie");
        app.UseCookieAuthentication(new CookieAuthenticationOptions
        {
            AuthenticationType = "ExternalCookie",
            AuthenticationMode = AuthenticationMode.Passive,
            CookieName = ".AspNet.ExternalCookie",
            ExpireTimeSpan = TimeSpan.FromMinutes(5),
        });

        var options = new GitHubAuthenticationOptions
        {
            ClientId = "your client id",
            ClientSecret = "your client secret",
            Provider = new GitHubAuthenticationProvider
            {
                OnAuthenticated = context =>
                {
                    context.Identity.AddClaim(new Claim("urn:token:github", context.AccessToken));

                    return Task.FromResult(true);
                }
            }
        };
        app.UseGitHubAuthentication(options);
    }
}
```

### Create a challenge result class

The next bit which is needed is to create a `ChallengeResult` class:

``` csharp
internal class ChallengeResult : HttpUnauthorizedResult
{
    public ChallengeResult(string provider, string redirectUri)
    {
        LoginProvider = provider;
        RedirectUri = redirectUri;
    }

    public string LoginProvider { get; set; }
    public string RedirectUri { get; set; }
    public override void ExecuteResult(ControllerContext context)
    {
        var properties = new AuthenticationProperties { RedirectUri = RedirectUri };
        context.HttpContext.GetOwinContext().Authentication.Challenge(properties, LoginProvider);
    }
}
```

The `ChallengeResult` class is a custom `ActionResult` which will tell the OWIN middleware that we will need to get authentication via the particular login provider.

### Putting it all together

Now we can put it all together. Add a new controller action to your `HomeController` which will return a `ChallengeResult` and ask the OWIN middleware that we need to get authentication from the `GitHub` provider. 

We also specify to the OWIN middleware that it needs to redirect to the `Index` action after the GitHub authentication has completed. 

``` csharp
public ActionResult AuthorizeGitHub()
{
    return new ChallengeResult("GitHub", Url.Action("Index"));
}
```

The next bit is to write the code for the `Index` action:

``` csharp
public async Task<ActionResult> Index()
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

    return View(userDisplayModel);
}
```

This method will get the authentication result from the OWIN middleware and check the `ClaimsIdentity` for the claim containing the GitHub token (which was set in the `OnAuthenticated` callback). If an authentication result is retrieved (i.e. we have authentication via GitHub already) and we are able to retrieve an access token, I simply use the access token to retrieve the information about the current user via the `GitHubClient` class. 

Finally I store all the data in a custom view model and pass it along to the view to be displayed to the user after they have authenticated with GitHub:

![](/assets/images/2015-04-27-using-aspnet-oauth-providers-without-identity/index-page.png)

## Conclusion

In this blog post I showed you how you can use the OWIN OAuth authentication providers without ASP.NET Identity. This is very handy in situations where you just want to retrieve an access token to make OAuth calls to a 3rd party service, but you do not want to use full blown ASP.NET Identity.
