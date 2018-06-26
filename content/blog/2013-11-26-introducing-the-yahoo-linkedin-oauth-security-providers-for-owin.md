---
date: 2013-11-26T00:00:00Z
description: |
  Introduces you to the Yahoo and LinkedIn OAuth providers for ASP.NET MVC (using OWIN) and shows you how to use them in your own application.
tags:
- katana
- nuget
- oauth
- owin
title: Introducing the Yahoo and LinkedIn OAuth security providers for OWIN
url: /blog/introducing-the-yahoo-linkedin-oauth-security-providers-for-owin/
---

**Update: 1 April 2015** - The latest and most up to date version of the **LinkedIn** guide [is available on the OAuth for ASP.NET website](http://www.oauthforaspnet.com/providers/linkedin/)

**Update: 1 April 2015** - The latest and most up to date version of the **Yahoo** guide [is available on the OAuth for ASP.NET website](http://www.oauthforaspnet.com/providers/yahoo/)

***

In [my previous blog bost](/blog/google-oauth-sign-asp-net-identity/) I wrote about the Google+ OAuth provider which I developed for ASP.NET Identity (OWIN).  This code has since been taken over by Microsoft and will be integrated into the Katana project.  I had some extra time on my hands and set about developing OAuth providers for LinkedIn and Yahoo, as they were apparently the ones which most people requested to be added.

I am glad to announce that these providers are now ready for you to use in your own projects.  In this blog post I will explain how to use these two security providers in your own application.  If you are not familiar with ASP.NET Identity I suggest you start of by reading the article entitled [Create an ASP.NET MVC 5 App with Facebook and Google OAuth2 and OpenID Sign-on](http://www.asp.net/mvc/tutorials/mvc-5/create-an-aspnet-mvc-5-app-with-facebook-and-google-oauth2-and-openid-sign-on) on the ASP.NET website.

## Installing the package via NuGet

![](/assets/images/2013/11/nuget.png)

The first step is to install the package via NuGet.  The name of the package is Owin.Security.Providers

![](/assets/images/2013/11/nuget2.png)

## Registering the Yahoo provider

To use the Yahoo OAuth provider you will need to create a project on the Yahoo Developer Network.  First step is to head over to [http://developer.yahoo.com/](http://developer.yahoo.com/) and register as a developer.  Once you are registered as a developer you will need to register a project to obtain a consumer key and secret.

Click on your name in the top right corner and select My Projects:

![](/assets/images/2013/11/owin1.png)

You will see a list of the existing projects which you have registered (if any).  Click on the Create Project button to register a new project.

![](/assets/images/2013/11/owin2.png)

Complete the new project form.  Make sure that under Application Type your select 'Web-based'.  Under Access Scopes select 'This app requires access to private user data' and be sure to **select at least one** of the APIs under 'Select APIs for private user data access'.  If you do not select at least one of the APIs your will get all sort of weird errors coming back from Yahoo, so just go ahead and pick one of the APIs.

![](/assets/images/2013/11/owin3.png)

Once you are done you can click on the Create Project button.  After the project is created you will be presented with a screen which will display your Consumer Key and Consumer Secret

![](/assets/images/2013/11/owin4.png)

Head on over to the Startup class located in App_Start\Startup.Auth.cs and add the namespace:

``` csharp
using Owin.Security.Providers.Yahoo;
```

Next register the Yahoo provider in the ConfigureAuth method:

``` csharp
app.UseYahooAuthentication("<YOUR CONSUMER KEY>", "<YOUR CONSUMER SECRET>");
```

## Register the LinkedIn provider

To use the LinkedIn provider you will also need to register an application on LinkedIn.  Head on over to [http://developer.linkedin.com](http://developer.linkedin.com/) and ensure that you are registered as a developer.  Click on your name in the top right corner and select API Keys:

![](/assets/images/2013/11/owin5.png)

You will be presented with a list of your current registered applications, if any.  Click on Add New Application.

![](/assets/images/2013/11/owin6.png)

Go ahead an complete all the required information.  Once done you will be presented with a screen which displays a summary of the new application, including the **API Key** and **Secret Key**.  Take note of those two as you will need them next when you register the LinkedIn security provider.

![](/assets/images/2013/11/owin7.png)

Once again  head on over to the Startup class located in App_Start\Startup.Auth.cs and add the namespace:

``` csharp
using Owin.Security.Providers.LinkedIn;
```

And finally register the LinkedIn provider in the ConfigureAuth method:

``` csharp
app.UseLinkedInAuthentication("<YOUR API KEY>", "<YOUR SECRET KEY>");
```


## Conclusion

This blog post showed you how to easily add external authentication using Yahoo and LinkedIn to your application.  The repository for the providers are located at [https://github.com/owin-middleware/OwinOAuthProviders](https://github.com/owin-middleware/OwinOAuthProviders), so if you experience any issues or if you want to add more providers please clone the repository and send a pull request.