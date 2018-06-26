---
title: "Implementing an OpenIddict Authorization server: A Basic Authorization Server"
description: |
  A basic introduction to creating an OpenIddict authorization server for OpenID Connect clients.
date: 2018-02-26
tags:
- asp.net
- asp.net core
- openiddict
- github
- authorization
url: /blog/implementing-openiddict-authorization-server-part-1
---

[OpenIddict](https://github.com/openiddict/openiddict-core) is an open source framework for ASP.NET Core which allows you to easily implement an [OpenID Connect](https://openid.net/connect/) server. In this blog post series, we will look at how you can implement social login with GitHub in your OpenIddict authorization server and create a simple Angular application which uses the Implicit Flow to authenticate a user.

Please note that even though some of the [official OpenIddict samples](https://github.com/openiddict/openiddict-samples) implement social login, they do so as part of using [ASP.NET Identity](https://docs.microsoft.com/en-us/aspnet/identity/overview/getting-started/introduction-to-aspnet-identity) as the underlying authentication mechanism. This series will demonstrate how you can do this independently of ASP.NET Identity.

## Create a new application

Let's start off with building a basic OpenIddict server. Create a new ASP.NET Core Web Application and ensure that you select the **Empty** template:

![](/assets/images/2018-02-26-implementing-openiddict-authorization-server-part-1/new-aspnet-core-application.png)

Once the application has been created, add the following NuGet packages to your application which are required by OpenIddict:

```text
dotnet add package AspNet.Security.OAuth.Validation --version 2.0.0-rc2-final 
dotnet add package OpenIddict --version 2.0.0-rc2-final 
dotnet add package OpenIddict.Mvc --version 2.0.0-rc2-final
dotnet add package OpenIddict.EntityFrameworkCore --version 2.0.0-rc2-final
```

## Add the MVC middleware

The **Empty** template creates a pretty barren ASP.NET Core application. We will need to configure the MVC middleware as OpenIddict is dependent on that, so go ahead and open the `Startup` class. You will need to update the `ConfigureServices` method to register the MVC services, and also update the `Configure` method to add the MVC middleware. This is what my `Startup` class looks like at this point:

```csharp
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddMvc();
    }

    public void Configure(IApplicationBuilder app, IHostingEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        app.UseMvcWithDefaultRoute();
    }
}
```

### Create a database context

OpenIddict will use Entity Framework to store certain information regarding its settings and operations, so create a database context with a constructor as per the sample snippet below:

```csharp
public class AuthorizationDbContext : DbContext
{
    public AuthorizationDbContext(DbContextOptions options) : base(options)
    {
    }
}
```

Register the EF services for the database context, configuring SQL Server as the underlying database. Also, notice the call to `UseOpenIddict()` which will add the entities required by OpenIddict to the database context:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContext<AuthorizationDbContext>(options =>
    {
        // Configure the context to use Microsoft SQL Server.
        options.UseSqlServer(Configuration.GetConnectionString("AuthorizationDbContext"));

        // Register the entity sets needed by OpenIddict
        options.UseOpenIddict();
    });

    services.AddMvcCore();
}
```

In the code sample above, the call to `UseSqlServer()` passed a connection string for the SQL Database, so be sure to add an `appsettings.json` file with a setting for the connection string specified:

```json
{
  "ConnectionStrings": {
    "AuthorizationDbContext": "Server=(localdb)\\MSSQLLocalDB;Database=OIDC_Authorization_Server;Trusted_Connection=True;MultipleActiveResultSets=true"
  }
}
```

### Create the database

We will need to create the actual database for the database context. For that we will first need to create the database migrations:

```text
dotnet ef migrations add CreateOpenIddictModels
```

And then execute the migrations:

```text
dotnet ef database update
```

Once that is done you will notice a set of OpenIddict tables added to your database which OpenIddict will use for its operation:

![](/assets/images/2018-02-26-implementing-openiddict-authorization-server-part-1/openiddict-database-tables.png)

### Register the middleware

Now, we will need to add all the service and middleware registrations required by OpenIddict. First, update the `ConfigureServices` method to register the authentication services. We will configure cookie authentication, as we will eventually store the user's GitHub authentication information inside a cookie. We also register the OAuth2 validation handler by calling `AddOAuthValidation()`. This will validate the tokens issued by OpenIddict.

The second part is to register all the OpenIddict services by calling `AddOpenIddict()`. Note the comments in the code snippet below for further explanations on what each of those lines do. The important bit, for now, is to note that I have configured the OIDC Implicit Flow which since I will implement an Angular application later in this series. 

Also, notice that I specified the path to the authorization endpoint as `/connect/authorize`:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContext<AuthorizationDbContext>(options =>
    {
        // code omitted for brevity...
    });

    services.AddMvc();

    services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
        .AddCookie()
        .AddOAuthValidation();

    services.AddOpenIddict(options =>
    {
        // Register the Entity Framework stores.
        options.AddEntityFrameworkCoreStores<AuthorizationDbContext>();

        // Register the ASP.NET Core MVC binder used by OpenIddict
        options.AddMvcBinders();

        // Enable the authorization endpoints
        options.EnableAuthorizationEndpoint("/connect/authorize");

        // Allow client applications to use the implicit flow.
        options.AllowImplicitFlow();

        // During development, you can disable the HTTPS requirement.
        options.DisableHttpsRequirement();

        // Register a new ephemeral key, that is discarded when the application
        // shuts down. Tokens signed using this key are automatically invalidated.
        // This method should only be used during development.
        options.AddEphemeralSigningKey();
    });
}    
```

At this point we can run the application and have a look at the [OpenId Connect Configuration information document](https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderConfig):

![](/assets/images/2018-02-26-implementing-openiddict-authorization-server-part-1/openid-connect-configuration.png)

## Create an authorization controller

The authorization server will not work at this point since we have not implemented the actual authorization endpoint. As noted before, we will eventually authenticate the users with their GitHub account, but for now, to close off this initial blog post, let's simply return a hard-coded user.

Create a `Controllers` folder and add an `AuthorizationController` class. We will create one action called `Authorize` which will handle that `/connect/authorize` route which we specified as the authorization endpoint for OpenIddict:

```csharp
public class AuthorizationController : Controller
{
    [HttpGet("~/connect/authorize")]
    public async Task<IActionResult> Authorize(OpenIdConnectRequest request)
    {
    }
}
```

Since we are returning a hard-coded user (or subject in OIDC terms), the code will be very simple. We will create an `AuthenticationTicket` and return a `SignInResult`:

```csharp
public class AuthorizationController : Controller
{
    [HttpGet("~/connect/authorize")]
    public async Task<IActionResult> Authorize(OpenIdConnectRequest request)
    {
        // Create a new ClaimsPrincipal containing the claims that
        // will be used to create an id_token, a token or a code.
        var claims = new List<Claim>();
        claims.Add(new Claim(OpenIdConnectConstants.Claims.Subject, "user-0001"));
        var identity = new ClaimsIdentity(claims, "OpenIddict");
        var principal = new ClaimsPrincipal(identity);

        // Create a new authentication ticket holding the user identity.
        var ticket = new AuthenticationTicket(principal,
            new AuthenticationProperties(), 
            OpenIdConnectServerDefaults.AuthenticationScheme);

        // Returning a SignInResult will ask OpenIddict to issue the appropriate access/identity tokens.
        return SignIn(ticket.Principal, ticket.Properties, ticket.AuthenticationScheme);
    }
}
```

## Configuring OpenIddict

One final bit to do is to configure a client in OpenIddict, as we will require a `client_id` when calling the authorization endpoint. For the client we will specify an ID, Name, as well as some redirect URIs.

I just created an `InitializeAsync` method in my `Startup` class which will seed the OpenIddict tables with the correct values, and then call this `InitializeAsync` method from the `Configure` method:

```csharp
public class Startup
{
    // Some code omitted for brevity...
    
    public void Configure(IApplicationBuilder app, IHostingEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        app.UseAuthentication();

        app.UseMvcWithDefaultRoute();

        InitializeAsync(app.ApplicationServices, CancellationToken.None).GetAwaiter().GetResult();
    }

    private async Task InitializeAsync(IServiceProvider services, CancellationToken cancellationToken)
    {
        // Create a new service scope to ensure the database context is correctly disposed when this methods returns.
        using (var scope = services.GetRequiredService<IServiceScopeFactory>().CreateScope())
        {
            var manager = scope.ServiceProvider.GetRequiredService<OpenIddictApplicationManager<OpenIddictApplication>>();

            if (await manager.FindByClientIdAsync("angular-app", cancellationToken) == null)
            {
                var descriptor = new OpenIddictApplicationDescriptor
                {
                    ClientId = "angular-app",
                    DisplayName = "Angular Application",
                    PostLogoutRedirectUris = { new Uri("https://oidcdebugger.com/debug") },
                    RedirectUris = { new Uri("https://oidcdebugger.com/debug") }
                };

                await manager.CreateAsync(descriptor, cancellationToken);
            }
        }
    }
}
```

## Testing it out

OK, this blog post is getting a bit on the long side, so let's test this out. [Nate Barbettini](https://www.recaffeinate.co/) created an OIDC Debugger website which will come in handy at this point. Head on over to [https://oidcdebugger.com/](https://oidcdebugger.com/) and specify the settings for your OIDC authorization server:

![](/assets/images/2018-02-26-implementing-openiddict-authorization-server-part-1/oidc-debugger.png)

>  You may have noticed in the database seeding code from the previous section, that I specified the `https://oidcdebugger.com/debug` endpoint for my redirect URIs. This is the debugging endpoint for the OIDC Debugger and will allow us to inspect the tokens that were issued.

Click the **Send Request** button at the bottom of the page:

![](/assets/images/2018-02-26-implementing-openiddict-authorization-server-part-1/oidc-debugger-send.png)

This will initiate the OIDC authorization flow and head off to our new authorization server. Since we hard coded the user, there is no actual screen at the moment which requires the user to sign in. Instead, our authorization will immediately call back to the redirect URI with the tokens, which can be seen in the screenshot below:

![](/assets/images/2018-02-26-implementing-openiddict-authorization-server-part-1/oidc-debugger-result.png)

In the [next blog post](/blog/implementing-openiddict-authorization-server-part-2) we'll look at allowing the user to sign in with their GitHub account.

## Source code

Source code for this series can be found at [https://github.com/jerriepelser-blog/openiddict-social-login](https://github.com/jerriepelser-blog/openiddict-social-login)