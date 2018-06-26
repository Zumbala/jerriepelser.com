---
date: 2015-04-14T00:00:00Z
description: |
  An introduction to the new generic OAuth authentication provider which is part of ASP.NET 5
tags:
- aspnet
- aspnet5
- oauth
- authentication
title: An introduction to the ASP.NET 5 Generic OAuth Provider
url: /blog/introduction-to-aspnet5-generic-oauth-provider/
---

## Introduction

Over the past 2 years I have been involved in the [OWin Authentication Providers](https://github.com/RockstarLabs/OwinOAuthProviders) project, which enables developers to allow users of their ASP.NET applications to sign in with a wide range of services such as GitHub, LinkedIn, Yahoo and others. 

It started off with a blog post on how to allow users to sign in with their Google+ accounts via OAuth, and then it went on to LinkedIn and Yahoo and before long a Nuget package was born and the number of providers grew to where it stands at 23 today.

A lot of that code was generic and shared between the multiple providers, but no effort was ever made from my side to attempt to create a unified base class or generic OAuth provider which people could easily configure to work with any service. 

This was something that almost every contributor to the library noticed and commented on.

Thankfully with ASP.NET 5 the guys from Microsoft have been developing a generic OAuth provider which can be easily configured to work with most services out there.

I am writing a couple of blog posts which will cover the following:
1. An overview of the new generic OAuth provider and how to configure it (this blog post)
2. [Some more advanced configuration scenarios](http://www.jerriepelser.com/blog/advanced-configuration-in-the-aspnet5-generic-oauth-provider)

I am writing these blog posts with Visual Studio CTP 6 and ASP.NET 5 Beta 3. Things are bound to change from here on in, so if you are reading this a few months down the line, hopefully things are still familiar enough for you to make sense of them.

## The OAuth Flow

Before we get going with the basic configuration, I would like to go through the OAuth 2 flow quickly so you can understand how things fit together.

In terms of terminology I will be referring to Consumers and Service Providers. A Consumer is the application who will be requesting an OAuth token, so for example your ASP.NET application. The Service Provider is the application or service which authorizes the user and issues the token. 

In this blog post I will demonstrate how to configure authentication with GitHub, so for the sake of this blog post think of GitHub as the Service Provider.

Before anything can be done you will need to register an application with the Service Provider. Typically you will need to specify a name for the application and a **redirect URI**. Once an application has been registered, the Service Provider will issue a **client ID** and a **client secret** which is used during the authentication and token request process.

As for the actual OAuth 2 flow, it looks as follows: 

![](/assets/images/2015-04-14-introduction-to-aspnet5-generic-oauth-provider/oauth-flow.png)

1. The Consumer makes a request to the Service Provider **authorization endpoint** the authorize the user.
2. The Service Provider authenticates the user and prompts them whether to authorize the Consumer to access their information.
3. If the user authorizes the Consumer, the Service Provider redirects back to the **redirect URI** on the Consumer's website with a temporary access code.
4. The Consumer calls the **token endpoint** on the Service Provider website to exchange the code for a more permanent access token.
5. The Service Provider grants an access token which can be used to authenticate subsequent requests to protected resources

Now, there are various little nuances between all the different Service Providers, but the flow described above covers the basics and holds true for most Service Providers. 

In the section above I have made the important parts you need when configuring the generic OAuth provider in **bold**.  They are:
1. The Client ID
2. The Client secret
3. The Redirect URI
4. The Authorization endpoint
5. The Token endpoint

The Client ID and secret are obtained when registering your application on the Service Provider's website. Some Service Providers may call them something else, like Facebook for example will refer to them as the App ID and App Secret. 

The Authorization and Token endpoints can be typically obtained from the developer documentation of the Service Provider. I will try and add this information at a later stage to my [OAuth for ASP.NET Website](http://www.oauthforaspnet.com/) for easy reference.

The redirect URI is something which you can specify when configuring the OAuth authentication on the Consumer. This needs to match up with whatever you specify when creating the application on the Service Provider. This will normally be referred to as either a **redirect URL** or **callback URL** or some sort of variation of that.

Using our example of GitHub, this information can be found on the OAuth section of the API documentation at https://developer.github.com/v3/oauth/

You will see they list the Authorization endpoint as

``` text
https://github.com/login/oauth/authorize
```

and the Token endpoint as

``` text
https://github.com/login/oauth/access_token
```

The Authorization endpoint will usually have the path `/authorize`, `/authenticate`, `/auth` or something similar. The Token endpoint will typically have the path `/access_token` or `/token`, so look out for those.

## Configuring the ASP.NET project

Since the default ASP.NET 5 project template does not yet have support for social logins built in, I am basing my code on the [SocialSample project](https://github.com/aspnet/Security/tree/dev/samples/SocialSample) in the ASP.NET Security repository on GitHub. I simply created an empty ASP.NET 5 project, and did a copy and paste exercise from the SocialSample project to bootstrap things correctly:

``` csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddDataProtection();
        services.Configure<ExternalAuthenticationOptions>(options =>
        {
            options.SignInAsAuthenticationType = CookieAuthenticationDefaults.AuthenticationType;
        });
    }

    public void Configure(IApplicationBuilder app)
    {
        app.UseCookieAuthentication(options =>
        {
            options.LoginPath = new PathString("/login");
        });

        // We'll configure GitHub over here...
		//...

        // Choose an authentication type
        app.Map("/login", socialApp =>
        {
            socialApp.Run(async context =>
            {
                string authType = context.Request.Query["authtype"];
                if (!string.IsNullOrEmpty(authType))
                {
                    context.Response.Challenge(new AuthenticationProperties() { RedirectUri = "/" }, authType);
                    return;
                }

                context.Response.ContentType = "text/html";
                await context.Response.WriteAsync("<html><body>");
                await context.Response.WriteAsync("Choose an authentication type: <br>");
                foreach (var type in context.GetAuthenticationTypes())
                {
                    await context.Response.WriteAsync("<a href=\"?authtype=" + type.AuthenticationType + "\">" + (type.Caption ?? "(suppressed)") + "</a><br>");
                }
                await context.Response.WriteAsync("</body></html>");
            });
        });

        // Sign-out to remove the user cookie.
        app.Map("/logout", socialApp =>
        {
            socialApp.Run(async context =>
            {
                context.Response.SignOut(CookieAuthenticationDefaults.AuthenticationType);
                context.Response.ContentType = "text/html";
                await context.Response.WriteAsync("<html><body>");
                await context.Response.WriteAsync("You have been logged out. Goodbye " + context.User.Identity.Name + "<br>");
                await context.Response.WriteAsync("<a href=\"/\">Home</a>");
                await context.Response.WriteAsync("</body></html>");
            });
        });

        // Deny anonymous request beyond this point.
        app.Use(async (context, next) =>
        {
            if (!context.User.Identity.IsAuthenticated)
            {
                // The cookie middleware will intercept this 401 and redirect to /login
                context.Response.Challenge();
                return;
            }
            await next();
        });

        // Display user information
        app.Run(async context =>
        {
            context.Response.ContentType = "text/html";
            await context.Response.WriteAsync("<html><body>");
            await context.Response.WriteAsync("Hello " + context.User.Identity.Name + "<br>");
            foreach (var claim in context.User.Claims)
            {
                await context.Response.WriteAsync(claim.Type + ": " + claim.Value + "<br>");
            }
            await context.Response.WriteAsync("<a href=\"/logout\">Logout</a>");
            await context.Response.WriteAsync("</body></html>");
        });
    }
}
```

The code above does the following:

1. Enable Cookie based authentication (which seems to be required for the external login providers) and specifies the login path as `/login`, so any request which requires authentication will redirect to that path.
2. Define a `/login` path which renders the list of configured authentication types and allows the user to select one by clicking on a link. 
3. When the user clicks on a link for an authentication type, it will do a request to the `/login` path again, but this time passing in the authentication type the user requested. This will return a challenge response which invokes the specified authentication type.
4. Define a `/logout` path which will log the user out
5. Checks whether the user is authenticated, and if not will issue a challenge response which will redirect the user to the `/login` route.
6. Finally, if the user is authenticated it will render out the user's name and whatever claims are associated with the user, for example the OAuth token.

Think of all this as the normal boilerplate code which the default ASP.NET application template will create for you in some form or another once the final versions of Visual Studio 2015 and ASP.NET 5 is released. What is important for this blog post is the part which comes next, and that is configuring OAuth authentication with GitHub. 

First thing I did is to register a new application in GitHub. For the callback URL of the application I specified the URL of my web application (in my case `http://localhost:35723`) with the path `/signin-github`. Here is a screenshot of the application in GitHub where you can see all the settings, as well as the Client ID and Client Secret:

![](/assets/images/2015-04-14-introduction-to-aspnet5-generic-oauth-provider/github-app-settings.png)

With all that in place I add the configuration code for the GitHub authentication in my `Configure()` method where I left the comment before:

``` csharp
app.UseOAuthAuthentication("GitHub", options =>
{
    options.ClientId = "8de23516870d7145b84e";
    options.ClientSecret = "0e3fd9808072907d1aa01df7e547ddbb6f6ec2fb";
    options.CallbackPath = new PathString("/signin-github");
    options.AuthorizationEndpoint = "https://github.com/login/oauth/authorize";
    options.TokenEndpoint = "https://github.com/login/oauth/access_token";
});
```

What the code does is the following:

1. Register a new authentication type called "GitHub" which is using the new generic OAuth authentication handler
2. Specifies the client ID and secret as supplied by GitHub
3. Specifies the callback path as `/signin-github`. This must match with the Authorization Callback URL (i.e. the redirect URI) which you specified in GitHub
4. Specifies the authorization and token endpoints as specified in the GitHub documentation

## Running the application

With all that in place we can run the application. First thing you will notice is the screen where I select the authentication type:

![](/assets/images/2015-04-14-introduction-to-aspnet5-generic-oauth-provider/select-authentication-type.png)

Click on the GitHub link which will make a request to the login URL with the `authtype` parameter specified as "GitHub". This will return a challenge response which will invoke the generic OAuth handler which in turn will handle the OAuth 2 authentication workflow.

You will be directed to the GitHub website where you need to sign in if you have not done so yet, and then authorize the application to access your information:

![](/assets/images/2015-04-14-introduction-to-aspnet5-generic-oauth-provider/github-authorization.png)

Once you click on Authorize Application, GitHub will redirect back to the **redirect URI** with the temporary access code, and the OAuth handler will in turn call the token endpoint to request a permanent token. 

At this point the user is authenticated and the token will be added to the list of claims for the user which can be viewed once you are redirected to the root URL for the website:

![](/assets/images/2015-04-14-introduction-to-aspnet5-generic-oauth-provider/display-token.png)

## Conclusion

This blog post gave you an overview of the general OAuth authorization flow, as well as how to configure the new generic OAuth authentication handler inside ASP.NET 5. In the next blog post I will delve deeper into some more advanced configuration scenarios.  
