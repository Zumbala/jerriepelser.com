---
date: 2013-10-18T00:00:00Z
description: |
  Demonstrates how to customize the login buttons in ASP.NET MVC 5 with the look and feel of the various Social Media Networks
tags:
- aspnet
- aspnet mvc
- oauth
title: Pretty social login buttons for ASP.NET MVC 5
url: /blog/pretty-social-login-buttons-for-asp-net-mvc-5/
---

**Update: 19 May 2015** - You can [view a more recent video version of this technique on my AspnetCasts YouTube channel](https://www.youtube.com/watch?v=kJiKEAaqnrY). It makes use of the [Social Buttons for Bootstrap](http://lipis.github.io/bootstrap-social/) CSS library. 

***

Earlier in 2013 I wrote  blog post explaining [how to customise the external login buttons in ASP.NET MVC 4](/blog/customizing-external-login-buttons-in-asp-net-mvc-4/).  Thanks to a couple of links to that blog post from some of the Microsoft websites, that particular blog post proved quite popular.  Now that MVC 5 has come around I was asked to update the blog post to work with the new ASP.NET Identity Framework.  I will once again cover the entire process of how to get "pretty" social login buttons in MVC 5, so you don't have to refer back to the previous blog post at all.

Before we start I suggest you go and read [Rick Anderson's](https://twitter.com/RickAndMSFT) article explaining [how to enable OAuth signin in MVC 5](http://www.asp.net/mvc/tutorials/mvc-5/create-an-aspnet-mvc-5-app-with-facebook-and-google-oauth2-and-openid-sign-on).

## Register OAuth providers

We start of by creating a new ASP.NET MVC 5 application and enabling the various OAuth providers we would like to use in the App_Start\Startup.Auth.cs

``` csharp
app.UseTwitterAuthentication(
    consumerKey: "x",
    consumerSecret: "x");

app.UseFacebookAuthentication(
    appId: "x",
    appSecret: "x");

app.UseGoogleAuthentication();
```

## Add Social Buttons CSS

The next step is to get a set of good looking social login buttons.  I decided to use the [Zocial CSS social buttons](http://zocial.smcllns.com/) which you can grab directly from the Github repository available at [http://github.com/samcollins/css-social-buttons/](http://github.com/samcollins/css-social-buttons/). Add them to the project by copying the CSS file and the font files into the Content folder of your project.

Next we need to add the CSS file to your CSS bundle, by updating the relevant lines in the App_Code\BundleConfig.cs file as follows:

``` csharp
bundles.Add(new StyleBundle("~/Content/css").Include(
            "~/Content/bootstrap.css",
            "~/Content/site.css",
            "~/Content/zocial.css"
            ));
```

To use the Zocial CSS social buttons you can use any HTML such as a, div, span, button etc. and add a CSS class of “zocial” as well as the class for the service that you want to style the button for.  For example the HTML code

``` html
<a href="#" 
    class="zocial facebook">
    Sign in with Facebook
</a>
```

will render the button

![](/assets/images/2013/02/image_thumb.png "image")

## Customizing ASP.NET MVC Output

The final step required is to customize the output of the partial view which was created by the ASP.NET MVC Internet project template.  Microsoft changed the standard template a bit from the way it worked in MVC 4, so this time around we only need to edit the _ExternalLoginsListPartial.cshtml file which is found under Views\Account.

_ExternalLoginsListPartial.cshtml and is used to display the list of available logins on the Login page as well as the Account Management page.  We alter this view by rendering the correct CSS class which will allow the Zocial CSS classed to render the button correctly.  We are lucky in that the provider names in ASP.NET MVC corresponds to the correct CSS classes in Zocial, but remember that CSS is case sensitive, so we force the provider name to lower case using AuthenticationType.ToLower():

``` html
<button 
   type="submit" 
   class="zocial @p.AuthenticationType.ToLower()" 
   id="@p.AuthenticationType" name="provider" 
   value="@p.AuthenticationType" 
   title="Log in using your @p.Caption account">@p.AuthenticationType</button>
```

With this simple change the login page will display the login buttons using the new style:

![](/assets/images/2013/10/oauth1.png)

On the Manage Account the list of login providers will also be displayed using the new style:

![](/assets/images/2013/10/oauth2.png)