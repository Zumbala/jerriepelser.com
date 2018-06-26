---
date: 2013-10-28T00:00:00Z
description: |
  You have authenticated a user in your ASP.NET application using their Twitter profile, but how do you store the Twitter OAuth token to make subsequent API calls? This shows you how.
tags:
- aspnet
- aspnet identity
- aspnet mvc
- oauth
- social media
- twitter
title: Get the Twitter profile image using ASP.NET Identity
url: /blog/get-the-twitter-profile-image-using-the-asp-net-identity/
---

ASP.NET has had its share of membership systems over the years and with the latest release of Visual Studio 2013 and ASP.NET MVC 5, Microsoft has introduced a new system yet again referred to as ASP.NET Identity.  For a good overview of ASP.NET Identity and Microsoft's rationale on developing yet another new identity system for ASP.NET, please go and read the post entitled [Introduction to ASP.NET Identity](http://www.asp.net/identity/overview/getting-started/introduction-to-aspnet-identity) by Pranav Rastogi.

ASP.NET Identity comes with full support for OAuth and OpenID login providers, and provides for the ability to log in using Twitter, Facebook, Google and Microsoft accounts out of the box.  This process is described in more details in Rick Anderson's blog post [Create an ASP.NET MVC 5 App with Facebook and Google OAuth2 and OpenID Sign-on](http://www.asp.net/mvc/tutorials/mvc-5/create-an-aspnet-mvc-5-app-with-facebook-and-google-oauth2-and-openid-sign-on).

In this blog post I will show you how to build on the basic techniques used in the 2 blog posts referenced above to allow a user to log in using their Twitter account and associate their Twitter profile image with their user account.

This blog post assumes that you have already registered your application on the Twitter developer website and have the Consumer Key and Consumer Secret available.  If not, [then please hop on over and do it now](https://dev.twitter.com/apps).

## Configuring the Twitter Authentication provider

First off we need to enable the Twitter authentication provider in the Startup class.  This is usually done by adding the following line:

``` csharp
app.UseTwitterAuthentication("key", "secret");
```

For our purposes however we need to expand on the registration of the Twitter authentication provider by adding an `OnAuthenticated` callback delegate:

``` csharp
app.UseTwitterAuthentication(
    new TwitterAuthenticationOptions()
    {
        ConsumerKey = "your consumer key here",
        ConsumerSecret = "your consumer secret here",
        Provider = new TwitterAuthenticationProvider()
        {
            OnAuthenticated = async context =>
            {
                context.Identity.AddClaim(new Claim("urn:tokens:twitter:accesstoken", context.AccessToken));
                context.Identity.AddClaim(new Claim("urn:tokens:twitter:accesstokensecret", context.AccessTokenSecret));
            }
        }
    }
    );
```

The OnAuthenticated delegate will be called by the Twitter provider after successful authentication and provide a place where the Twitter access token and access token secret can be retrieved.  These are then added as claims to the ClaimsIdentity which is available as Identity property of the context variable.  Note that we are prefixing the type name of the access token related claims with "urn:tokens:" as this will make it a bit easier for us to retrieve them later on from the list of claims.

This is however only part of the solution, and to have the token and secret available for future reference they will need to be stored in the database.

## Storing the Twitter access token and secret

The way the normal authentication flow works in ASP.NET MVC 5 is that when you log on using a social media account such as Twitter, the AccountController will check to see if a user is already registered against that Twitter account.  If the user is registered they simply get logged on, but it it is the first time they log on they will be prompted to supply a user name.  Upon completion of the user name form, a postback will happen to the `ExternalLoginConfirmation()` method on the AccountController class.

What we need to do is to add code at this point which stores the claims containing the Twitter access token and secret and store it against the user profile in the database.  The first point is to create a new method to retrieve and store the tokens:

``` csharp
private async Task StoreAuthTokenClaims(ApplicationUser user)
{
    // Get the claims identity
    ClaimsIdentity claimsIdentity =
        await AuthenticationManager.GetExternalIdentityAsync(DefaultAuthenticationTypes.ExternalCookie);

    if (claimsIdentity != null)
    {
        // Retrieve the existing claims
        var currentClaims = await UserManager.GetClaimsAsync(user.Id);

        // Get the list of access token related claims from the identity
        var tokenClaims = claimsIdentity.Claims
            .Where(c => c.Type.StartsWith("urn:tokens:"));

        // Save the access token related claims
        foreach (var tokenClaim in tokenClaims)
        {
            if (!currentClaims.Contains(tokenClaim))
            {
                await UserManager.AddClaimAsync(user.Id, tokenClaim);
            }
        }
    }
}
```

In line 4 we retrieve the ClaimsIdentity so that we can access the claims which were added by the Twitter authentication provider in the OnAuthenticated callback delegate.  Line 10 retrieves the existing claims which is stored for the user as we want to check against these and only add them if they do not already exist.  Line 13 grabs the list of access token related claims - remember that we prefixed them before using "urn:tokens:" as we are only interested in storing those particular ones in the database.  Lines 17 - 23 iterates through the list of access token related claims and if they do not already exist for the user we store them against the user in the database by making a call to UserManager.AddClaimAsync() method.  The call to UserManager.AddClaimAsync() will store the claim in the database table called AspNetUserClaims, and because of this we can easily retrieve them at any time again later.

Of course we need to call this method, so simply alter the relevant lines in the `ExternalLoginConfirmation` method of the `AccountController` class:

``` csharp
result = await UserManager.AddLoginAsync(user.Id, info.Login);
if (result.Succeeded)
{
    await StoreAuthTokenClaims(user);

    // Sign in and redirect the user
    await SignInAsync(user, isPersistent: false);
    return RedirectToLocal(returnUrl);
}
```

Ensure that you call the method after the user has been created and the login has been associated against the user., but before you sign them in and redirect them.

To retrieve the claims from now on is as simple as adding the following lines of code:

``` csharp
var claimsIdentity = HttpContext.User.Identity as ClaimsIdentity;
if (claimsIdentity != null)
   var claims = claimsIdentity.Claims;
```

## Retrieving the profile image

I promised you that we will retrieve the Twitter profile image for the user and so far we haven't done that.  The ground work has been laid however, and downloading the profile image from Twitter is all that remains:

``` csharp
private async Task DownloadTwitterProfileImage(IEnumerable<Claim> claims, string userId)
{
    // Retrieve the twitter access token and claim
    var accessTokenClaim = claims.FirstOrDefault(x => x.Type == "urn:tokens:twitter:accesstoken");
    var accessTokenSecretClaim = claims.FirstOrDefault(x => x.Type == "urn:tokens:twitter:accesstokensecret");

    if (accessTokenClaim != null && accessTokenSecretClaim != null)
    {
        // Initialize the Twitter client
        var service = new TwitterService(
            "your twitter consumer key",
            "your twitter consumer secret",
            accessTokenClaim.Value,
            accessTokenSecretClaim.Value
            );

        var profile = service.GetUserProfile(new GetUserProfileOptions());
        if (profile != null && !String.IsNullOrWhiteSpace(profile.ProfileImageUrlHttps))
        {
            string filename = Server.MapPath(string.Format("~/ProfileImages/{0}.jpeg", userId));

            await DownloadProfileImage(new Uri(profile.ProfileImageUrlHttps), filename);
        }
    }
}
```

Lines 4 + 5 simply retrieves the claims for the Twitter access token and secret.  On line 10 create an instance on the `TwitterService` class passing in your consumer key and secret, as well as the access token and secret which was retrieved from the claims.  (The TwitterService class is from the [TweetSharp library](http://www.nuget.org/packages/TweetSharp/) which I installed via Nuget).  Line 17 makes the call to Twitter to retrieve the user profile.  The **TwitterUser** class which is returned by the call to GetUserProfile() contains a property called **ProfileImageUrlHttps** which is the URL to user's profile image.  The rest of the code simply calls another helper method which downloads the image from the URL and saves it in a folder call "/ProfileImages/" with the same file name as the user's id.

A call is added to this method in the our `StoreAuthTokenClaims` method from above:

``` csharp
// Save the access token related claims
foreach (var tokenClaim in tokenClaims)
{
    if (!currentClaims.Contains(tokenClaim))
    {
        await UserManager.AddClaimAsync(user.Id, tokenClaim);
    }
}

// Download the twitter profile image
await DownloadTwitterProfileImage(tokenClaims, user.Id);
```

**Just a quick note before I carry on.**  Even though I stated before that you retrieve the claims from here on in from HttpContext.User.Identity, at this specific point we cannot do it as user has actually not been signed in yet, so we just pass the claims we retrieved before to the DownloadTwitterProfileImage() method.  Once the user has been signed in however (which happens just after we have stored the tokens and retrieved the image), your will be able to access the claims from HttpContext.User.Identity.

So, now that the profile image is downloaded to the server, all we need to do is display that image for the logged in user.  For that I add a small piece of HTML to the **_LoginPartial.cshtml** partial view.

``` html
<li>
    <img src="~/ProfileImages/@(User.Identity.GetUserId()).jpeg" />
</li>
```

And the final product is below with the user profile image displayed.  Since we store the OAuth token and secret we of course have access to the entire Twitter API, so in the screenshot below you will notice that I also retrieve and display the list of tweets from the user's Twitter home timeline.

![](/assets/images/2013/10/Capture4.png)

## What is missing

**Be warned: this is not production quality code!**.  There are a lot of places where things can go wrong and a lot of stuff which I hard coded.  This code was written to demonstrate one single concept.  Use it it your own systems as you wish, but it is up to you to make the code rock solid.

You may also have noticed that the solution provided above has a number flaws, and you will need to fix those on your own.  Off the top of my head, here are a few things you will need to think about:

1.  When removing the Twitter login from the account using the User Administration screen, you will need to remove the access token and secret from the claims table
2.  When associating a Twitter account using the User Administration screen, you will need to also retrieve and store the claims for the token at that point.  Look at the LinkLoginCallback(...) method of the AccountController class.
3.  In reality a user can associate multiple Twitter accounts to their user account - you will need to think a bit of how to handle that situation to differentiate the claims per associated login.
4.  I currently have this working only for Twitter.  If you want to do this for other providers such as Facebook you will need to add some logic to determine which login provider was used and download the profile image for that login provider.

## Source Code

There are quite a few parts to this solution and building it up progressively over the course of the blog post may leave you a bit confused.  Best is to have a look at the source for the complete final solution.  The relevant parts are:

*   Configuration of the Twitter authentication provider in the ConfigureAuth() method in the Startup.Auth.cs file.  **Also note **that you will need to enter your own Twitter consumer key and secret in this file.
*   Storing the claims and retrieving the user image in the AccountController class.  The relevant method are ExternalLoginConfirmation(), StoreAuthTokenClaims(), DownloadTwitterProfileImage() and DownloadProfileImage().
*   I have changed the Index() method on the HomeController to retrieve and display the latest Tweets for a logged in user.  Look at that the see how you can access the Twitter access token and secret at any stage from the ClaimsIdentity class.
