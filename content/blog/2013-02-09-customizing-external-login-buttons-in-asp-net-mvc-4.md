---
date: 2013-02-09T00:00:00Z
description: |
  Demonstrates how to customize the login buttons in ASP.NET MVC 4 with the look and feel of the various Social Media Networks.
tags:
- aspnet
- aspnet mvc
- oauth
title: Customizing External Login Buttons in ASP.NET MVC 4
url: /blog/customizing-external-login-buttons-in-asp-net-mvc-4/
---

**Update: 19 May 2015** - You can [view a more recent video version of this technique on my AspnetCasts YouTube channel](https://www.youtube.com/watch?v=kJiKEAaqnrY). It is for ASP.NET MVC 5, and makes use of the [Social Buttons for Bootstrap](http://lipis.github.io/bootstrap-social/) CSS library. 

**Update: 18 October 2013** - An updated version of the article for use with ASP.NET MVC 5 can be found at [/blog/pretty-social-login-buttons-for-asp-net-mvc-5/](/blog/pretty-social-login-buttons-for-asp-net-mvc-5/)

***

With the introduction of ASP.NET MVC 4 Microsoft has added functionality to allow you to easily add the ability for your users to log in to your websites using the existing their Facebook, Twitter, Google and other accounts using OAuth.  The standard functionality which is included in the default ASP.NET MVC 4 templates is pretty bland and does not apply any of the branding of the OAuth services you are using.  In this blog entry I will demonstrate how to easily add the appropriate branding to the login buttons for the various OAuth services.

To get more details on exactly how to enable OAuth authentication in your website please read the blog entry entitled [OAuth/OpenID Support for WebForms, MVC and WebPages](http://blogs.msdn.com/b/webdev/archive/2012/08/15/oauth-openid-support-for-webforms-mvc-and-webpages.aspx) first.

## Register OAuth providers

We start of by creating a new ASP.NET MVC 4 Internet Application and enabling the various OAuth providers we would like to use in the App_Start\AuthConfig.cs

``` csharp
OAuthWebSecurity.RegisterTwitterClient(
    consumerKey: "x",
    consumerSecret: "x");

OAuthWebSecurity.RegisterFacebookClient(
    appId: "x",
    appSecret: "x");

OAuthWebSecurity.RegisterGoogleClient();

OAuthWebSecurity.RegisterLinkedInClient(
    consumerKey: "x",
    consumerSecret: "x");

OAuthWebSecurity.RegisterYahooClient();
```

## Add Social Buttons CSS

The next step is to get a set of good looking social login buttons.  I decided to use the [Zocial CSS social buttons](http://zocial.smcllns.com/) which you can grab directly from the Github repository available at [https://github.com/gustavohenke/zocial-less](https://github.com/gustavohenke/zocial-less "https://github.com/gustavohenke/zocial-less"). Add them to the project by copying the CSS file and the font files into the Content folder of your project and add the CSS file to your App_Code\BundleConfig.css file

``` csharp
bundles.Add(new StyleBundle("~/Content/css").Include(
    "~/Content/site.css",
    "~/Content/zocial.css"));
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

The final step required is to customize the output of the partial views which was created by the ASP.NET MVC Internet project template.  There are two views which we need to edit and both can be found under Views\Account.

The first one is _ExternalLoginsListPartial.cshtml and is used to display the list of available logins on the Login page as well as the Account Management page.  We alter this view by rendering the correct CSS class which will allow Zocial to render the button correctly.  We are lucky in that the provider names in ASP.NET MVC corresponds to the correct CSS classes in Zocial, but remember that CSS is case sensitive, so we force the provider name to lower case using AuthenticationClient.ProviderName.ToLower():

``` html
<button 
    type="submit" 
    name="provider" 
    value="@p.AuthenticationClient.ProviderName" 
    class="zocial @p.AuthenticationClient.ProviderName.ToLower()" 
    title="Log in using your @p.DisplayName account">
    @p.DisplayName
</button>
```

The second view is _RemoveExternalLoginsPartial.cshtml which displays the existing services linked to a user's account.  Here we also need to render the correct CSS class for Zocial to do its magic.  This view simply renders the name if the service inside a TD, so we surround it with as SPAN with the correct class attribute:

``` html
<span class="zocial @externalLogin.ProviderDisplayName.ToLower()">
    @externalLogin.ProviderDisplayName
</span>
```

The final result of the Manage Account page with the correct styling applied can be view below:

![](/assets/images/2013/02/Capture4.png)