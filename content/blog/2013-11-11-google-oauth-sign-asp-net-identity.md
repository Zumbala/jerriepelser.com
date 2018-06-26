---
date: 2013-11-11T00:00:00Z
description: |
  ASP.NET MVC 5 does not include an OAuth login provider for Google+ out of the box. This post shows how you can create your own.
tags:
- aspnet identity
- aspnet mvc
- google+
- oauth
title: Google+ OAuth Sign In for ASP.NET Identity
url: /blog/google-oauth-sign-asp-net-identity/
---

**Update: 19 May 2015** - You can [view an updated video version of this on my AspnetCasts YouTube channel](https://www.youtube.com/watch?v=AZu3Q-ps_BY). The updated version is targeted for use with ASP.NET MVC 5 and ASP.NET Identity.

**Update: 1 April 2015** - The latest and most up to date version of this guide [is available on the OAuth for ASP.NET website](http://www.oauthforaspnet.com/providers/google/)

***

A few months ago I wrote a blog post on [how to add a Google+ sign in for ASP.NET MVC 4](/blog/google-signin-for-asp-net-mvc/).  With the release ASP.NET MVC 5 the whole authentication system has been overhauled and the ASP.NET team has [introduced ASP.NET Identity](http://www.asp.net/identity/overview/getting-started/introduction-to-aspnet-identity), which has built-in support for external authentication providers.  The list of external authentication providers which are included out of the box includes Google, but this is based on OpenID and not OAuth, which is fine if all you want to use it for is authentication.  Sadly however this is problematic if you want to use the user's authentication token to make calls to the Google+ API - for that you will need a Google+ OAuth provider.   Since no one else seems to have taken up the call to develop a Google+ OAuth provider for ASP.NET Identity, I decided to see what I can do.

I like the path of least resistance, and do not suffer too badly from the "[not invented here](http://en.wikipedia.org/wiki/Not_invented_here)" syndrome. The code for the Google+ OAuth provider I wrote is almost an exact copy of the code developed by the OWIN team for the standard Facebook OAuth provider.  I do not claim any level of cleverness or credit for this code, other than shamelessly copying it and adapting it to work with Google+

The original work from Microsoft is located at [http://katanaproject.codeplex.com/](http://katanaproject.codeplex.com/).  As it is open source I will see what I can do to integrate my work into theirs.  For now I just wanted to get something out which people can use.


## A quick rundown of what I have done

This blog post is actually a combination of a bunch of stuff I did in previous blog posts.  As noted above there was absolutely no genius involved on my side as far as the development of the actual OWIN OAuth provider goes.  I copied the work which Microsoft had done on the Facebook provider and adapted it for Google+.  Having done a LOT of OAuth work over the past year I understand more or less how the whole OAuth process works (I think).  I have walked through the Google+ OAuth sign in process in [a previous blog post](/blog/google-signin-for-asp-net-mvc/) so if you have a need to understand that please go and read that blog post.  Also have a look at [Google's own documentation](https://developers.google.com/accounts/docs/OAuth2Login) on how the OAuth 2.0 flow works.

To give the button the Google+ look and feel, you can refer to my blog post on [Pretty social login buttons for ASP.NET MVC 5](/blog/pretty-social-login-buttons-for-asp-net-mvc-5/).

Last month I wrote a blog post on how to access the Twitter OAuth token, and I used the same technique in the demo project to store and retrieve the Google+ OAuth token.  I then use this token on the home page to extract the Google+ user information and do a raw dump of the JSON output.  It is not pretty but it should give you a good enough idea of how to store and extract the OAuth token and then use that to call the Google+ API.

## How to use it

Well, to use it is pretty straight forward.   Here are the steps:

1. First of all you will need a Client ID and Client Secret and for that you need to register an app in the [Google Cloud Console](https://cloud.google.com/console).  I am not going to run through this process step by step, so please refer to the [Google documentation](https://developers.google.com/console/help/new/) for that.  You can also refer to that [previous blog post](/blog/google-signin-for-asp-net-mvc/) of mine for a more step-by-step walk through.  That blog post described how to do it in Google API Console, which is now replaced with the Google Cloud Console, but it should be similar enough to follow.

Please note that you will need to register your **Redirect URI** in Google Cloud Console to point to the URL **signin-googleplus**, as displayed in the screenshot below:

![](/assets/images/2013/11/Capture2.png)

2. Next you need to install the NuGet package I published into your project:

![](/assets/images/2013/11/Capture.png)

3.  Lastly alter your Startup.Auth.cs file to enable the Google+ authentication provider.

``` csharp
app.UseGooglePlusAuthentication("YOUR CLIENT ID ", "YOUR CLIENT SECRET");
```
