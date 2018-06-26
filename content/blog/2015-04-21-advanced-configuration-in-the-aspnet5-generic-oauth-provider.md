---
date: 2015-04-21T00:00:00Z
description: |
  I cover a couple of more advanced configuration scenarios in the ASP.NET 5 generic OAuth authentication provider, such as Scopes and retrieving user information.
tags:
- aspnet
- aspnet5
- oauth
- authentication
title: Advanced configuration in the ASP.NET 5 Generic OAuth Provider
url: /blog/advanced-configuration-in-the-aspnet5-generic-oauth-provider/
---

## Introduction

In my [previous blog post](http://www.jerriepelser.com/blog/introduction-to-aspnet5-generic-oauth-provider) I introduced the new generic OAuth 2 authentication provider which has been added to ASP.NET 5 by showing you how to configure it to authenticate with GitHub. In this blog post I will discuss two more advanced configuration options, namely requesting extra permissions via the Scope, and retrieving user information after authorization.

The code I use in this blog post is mostly a straight forward copy-and-paste exercise from [the sample in the ASP.NET Security repo on GitHub](https://github.com/aspnet/Security/tree/775eb5ece452e10a27aa166f52b58a6c0b615d18/samples/SocialSample).

## Request permissions via Scope

In OAuth terms the Scope is a means by which an OAuth consumer can tell the Service Provider the permissions it wants to request. The Service Provider will in turn inform the user what permissions the consumer is requesting, so the user can determine whether they want to grant permissions or not.

Once the user has granted permission, the scope is linked to the access token which is issued, so when a consumer uses that access token in the future to request protected data, it will not be allowed to access data outside of the scope which was granted.  

Some Service Providers do not implement a Scope at all, and allow consumers free reign as to what data it wants to read or update. Other Service Providers provide an extremely fine grained level op permissions through the scope. If you intend to use the access token for anything more than just authentication you should familiarize yourself with the allowed scopes of the Service Provider to ensure you request the correct permissions upfront.

The GitHub API [provides a fairly wide range of values for the scope](https://developer.github.com/v3/oauth/#scopes), allowing a consumer to be very specific in term of what actions it wants to perform. In this example I will request the `admin:org` scope, which means I intend to administer the user's organizations.

Requesting a scope is straight forward. The `OAuthAuthenticationOptions` class contains a `Scope` property which contains the list of scopes:

``` csharp
public class OAuthAuthenticationOptions : AuthenticationOptions
{
	...

	/// <summary>
	/// A list of permissions to request.
	/// 
	/// </summary>
	public IList<string> Scope { get; private set; }

	...
}
```

So to request the `admin:org` you simply add it to the list of Scopes during registration:

``` csharp
app.UseOAuthAuthentication("GitHub", options =>
{
    options.ClientId = "your client id";
    options.ClientSecret = "your client secret";
    options.CallbackPath = new PathString("/signin-github");
    options.AuthorizationEndpoint = "https://github.com/login/oauth/authorize";
    options.TokenEndpoint = "https://github.com/login/oauth/access_token";
    options.Scope.Add("admin:org");
}
```

Now when the user is prompted to allow our application permissions to access their information, you will see that GitHub is very explicit in indicating to the user that our application intends to administer organizations on their behalf:

![](/assets/images/2015-04-21-advanced-configuration-in-the-aspnet5-generic-oauth-provider/github-permissions.png)

So if your intention is to only authenticate the user, and not to use the access token afterwards to access any other information, be sure to request the minimum number of Scopes possible. Usually this means just leaving the `Scope` property empty. Requesting too many permissions may prevent the user from grating those permissions to your application.

## Requesting the user's information

At this point, once a user authenticates with GitHub, we will only get back an access token. But typically in an authentication scenario you would also like to get back some more information about the user. Information such as their name, email address etc.

The generic OAuth 2 provider has a `Notifications` property which allows you to hook into certain events during the authentication process. One of those is the `OnGetUserInformationAsync` delegate which gets called after a user has authenticated but before the actual `ClaimsIdentity` gets returned. This allows us to retrieve some extra information about the user.

Here the full code after I implementated `OnGetUserInformationAsync`:

``` csharp
app.UseOAuthAuthentication("GitHub", options =>
{
    options.ClientId = "your client id";
    options.ClientSecret = "your client secret";
    options.CallbackPath = new PathString("/signin-github");
    options.AuthorizationEndpoint = "https://github.com/login/oauth/authorize";
    options.TokenEndpoint = "https://github.com/login/oauth/access_token";
    options.UserInformationEndpoint = "https://api.github.com/user";
    options.Notifications = new OAuthAuthenticationNotifications
    {
        OnGetUserInformationAsync = async context =>
        {
            // Get the GitHub user
            HttpRequestMessage userRequest = new HttpRequestMessage(HttpMethod.Get, context.Options.UserInformationEndpoint);
            userRequest.Headers.Authorization = new AuthenticationHeaderValue("Bearer", context.AccessToken);
            userRequest.Headers.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
            HttpResponseMessage userResponse = await context.Backchannel.SendAsync(userRequest, context.HttpContext.RequestAborted);
            userResponse.EnsureSuccessStatusCode();
            var text = await userResponse.Content.ReadAsStringAsync();
            JObject user = JObject.Parse(text);

            var identity = new ClaimsIdentity(
                context.Options.AuthenticationType,
                ClaimsIdentity.DefaultNameClaimType,
                ClaimsIdentity.DefaultRoleClaimType);

            JToken value;
            var id = user.TryGetValue("id", out value) ? value.ToString() : null;
            if (!string.IsNullOrEmpty(id))
            {
                identity.AddClaim(new Claim(ClaimTypes.NameIdentifier, id, ClaimValueTypes.String, context.Options.AuthenticationType));
            }
            var userName = user.TryGetValue("login", out value) ? value.ToString() : null;
            if (!string.IsNullOrEmpty(userName))
            {
                identity.AddClaim(new Claim(ClaimsIdentity.DefaultNameClaimType, userName, ClaimValueTypes.String, context.Options.AuthenticationType));
            }
            var name = user.TryGetValue("name", out value) ? value.ToString() : null;
            if (!string.IsNullOrEmpty(name))
            {
                identity.AddClaim(new Claim("urn:github:name", name, ClaimValueTypes.String, context.Options.AuthenticationType));
            }
            var link = user.TryGetValue("url", out value) ? value.ToString() : null;
            if (!string.IsNullOrEmpty(link))
            {
                identity.AddClaim(new Claim("urn:github:url", link, ClaimValueTypes.String, context.Options.AuthenticationType));
            }

            context.Identity = identity;
        }
    };
});
```

In the code above you will see that I added a value for the `UserInformationEndpoint` property, which is just the URL for the [GitHub API endpoint](https://developer.github.com/v3/users/#get-the-authenticated-user) through which I will retrieve the user information.

I then proceed to specify an `OAuthAuthenticationNotifications` object which contains an implementation for the `OnGetUserInformationAsync` method. Inside my `OnGetUserInformationAsync` I simply make a request through to the `UserInformationEndpoint` which will return the user information in JSON format. I then extract certain pieces of information from that JSON document and store them as claims on the `ClaimsIdentity`, and then assign the identity to the Identity property of the context.

When we run the application again, you will see the information which was retrieved from GitHub was added to the list of claims:
 
![](/assets/images/2015-04-21-advanced-configuration-in-the-aspnet5-generic-oauth-provider/claims.png)

Just a couple of things I want to note:

1. If you specify an implementation for `OnGetUserInformationAsync` you **have** to create an instance of ClaimsIndentity and assign it to the `Identity` property of the context. The [default implementation of OnGetUserInformationAsync](https://github.com/aspnet/Security/blob/775eb5ece452e10a27aa166f52b58a6c0b615d18/src/Microsoft.AspNet.Authentication.OAuth/OAuthAuthenticationDefaults.cs) does that for you, and if you specify a handler for `OnGetUserInformationAsync` without setting the Identity, then the authentication will fail.
2. You will see that I specified the `UserInformationEndpoint` and then used the value of that property inside `OnGetUserInformationAsync` to make the call to GitHub. So far, from what I have seen, nothing else accesses that property,  so if you don't call it yourself, it is not going to be called anywhere else. So you can really just specify the URL directly inside `OnGetUserInformationAsync` without having to access it from the `UserInformationEndpoint` property if you want to.

## Conclusion

In this blog post I demonstrated how to request multiple scopes from a Service Provider through the generic OAuth 2 provider as well as how to request user information from the Service Provider. 
