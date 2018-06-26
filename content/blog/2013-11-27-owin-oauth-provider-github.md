---
date: 2013-11-27T00:00:00Z
description: |
  Introduces you to the GitHub OAuth provider for ASP.NET MVC (using OWIN) and shows you how to use it in your own projects.
tags:
- github
- katana
- oauth
- owin
title: OWIN OAuth provider for GitHub
url: /blog/owin-oauth-provider-github/
---

**Update: 19 May 2015** - You can [view an updated video version of this on my AspnetCasts YouTube channel](https://www.youtube.com/watch?v=krMILfdOu0w). The updated version is targeted for use with ASP.NET MVC 5 and ASP.NET Identity.

**Update: 1 April 2015** - The latest and most up to date version of this guide [is available on the OAuth for ASP.NET website](http://www.oauthforaspnet.com/providers/github/)

***

Turns out I had some extra time on my hands - or maybe I am just procrastinating.  Either way I have extended the authentication providers for OWIN which [I wrote about yesterday](/blog/introducing-the-yahoo-linkedin-oauth-security-providers-for-owin/) to now also include a provider for GitHub.  Not sure how much this will be used, but sometimes I like to write code just because I can.  The mere act of coding gives me pleasure :)  In any case, if you are developing an application which is used by developers then this may come in handy.

If you are not familiar with ASP.NET Identity I suggest you start of by reading the article entitled [Create an ASP.NET MVC 5 App with Facebook and Google OAuth2 and OpenID Sign-on](http://www.asp.net/mvc/tutorials/mvc-5/create-an-aspnet-mvc-5-app-with-facebook-and-google-oauth2-and-openid-sign-on) on the ASP.NET website.

First of all you will need to install the NuGet package, so:

![](/assets/images/2013/11/nuget.png)

## Registering application in GitHub

In order to use GitHub as an OAuth provider you will first need to register an application in GitHub.

Log in to GitHub and click on the **Account Settings** icon next to your profile

![](/assets/images/2013/11/owin11.png)

On the navigation bar select **Applications** and click on **Register new Application**.

![](/assets/images/2013/11/owin31.png)

Complete the information for your application.  **Please note** however that your will need to supply the correct **Authorization Callback URL**.  To do that your will need to specify the root URL for your application with the path **signin-github** appended to it, as displayed in the screenshot below.  It is important that you specify the correct URL as GitHub will validate the callback URL it receives from the OWIN authentication middleware against what you specified when you registered your application.

![](/assets/images/2013/11/owin51.png)

Once you have completed the required fields you can click on **Register Application**.  GitHub will display the summary of the application, along with the **Client ID** and **Client Secret**.  Take note of these two fields as you will need them to register the GitHub authentication provider below.

![](/assets/images/2013/11/owin61.png)

##  Register the provider

To register the provider in your MVC 5 application you would have had to install the NuGet package already.  The next step is to head on over to App_Start\Startup.Auth.cs file and make sure that you include the namespace for the provider.

``` csharp
using Owin.Security.Providers.GitHub;
```

And finally register the provider in the **ConfigureAuth** method using the **Client ID** and **Client Secret** supplied by GitHub:

``` csharp
app.UseGitHubAuthentication("<YOUR CLIENT ID>", "<YOUR CLIENT SECRET>");
```

## The code

The code for the provider is located at [https://github.com/owin-middleware/OwinOAuthProviders](https://github.com/owin-middleware/OwinOAuthProviders).  If you experience any issues, please fork, fix and send me a pull request.  Also feel free to use the same repository if you would like to add providers for other services.  Anyone up for adding VKontakte or Sina Weibo?
