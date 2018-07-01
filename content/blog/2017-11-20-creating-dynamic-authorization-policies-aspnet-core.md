---
title: "Creating Authorization Policies dynamically with ASP.NET Core"
description: |
  A great tip I picked up from an NDC Oslo presentation which shows how you can dynamically create authorization policies for the ASP.NET Core authorization middleware.
date: 2017-11-20
tags:
- asp.net 
- asp.net core
- jwt
- authorization
url: /blog/creating-dynamic-authorization-policies-aspnet-core
---

ASP.NET Core contains a [DefaultAuthorizationPolicyProvider](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.authorization.defaultauthorizationpolicyprovider?view=aspnetcore-2.0) class which resolves [authorization policies](https://docs.microsoft.com/en-us/aspnet/core/security/authorization/policies) at runtime. I was watching a recording of the [Implementing Authorization for Applications and APIs](https://www.youtube.com/watch?v=EJeZ3YNnqz8) talk from NDC Oslo by [Dominick Baier](https://leastprivilege.com/) and [Brock Allen](https://brockallen.com/) and saw a technique they demonstrated to resolve authorization policies dynamically at runtime.

I did an internet search and could not find this documented anywhere, so in this blog post I will explain how to do this. However, full credit for this goes to Dominick and Brock. If you want to hear the explanation from them directly, [watch from 36:05 in the video](https://www.youtube.com/watch?v=EJeZ3YNnqz8&feature=youtu.be&t=36m5s).

## Background

One place where this will be extremely useful is when checking whether an access token has a specific scope. In the [Auth0 ASP.NET Core API Quickstart](https://auth0.com/docs/quickstart/backend/aspnet-core-webapi) for example, we have a section which demonstrates how to restrict calling a particular API endpoint by checking whether the [access token being passed in contains a particular scope](https://auth0.com/docs/quickstart/backend/aspnet-core-webapi/v2/01-authorization#configure-the-scopes). 

As an example, let's say that I have an access token issued by Auth0 with the following payload:

```json
{
  "iss": "https://jerrie.auth0.com/",
  "sub": "auth0|58b3...",
  "aud": [
    "https://api.mycompany.com/messages",
    "https://jerrie.auth0.com/userinfo"
  ],
  "iat": 1511159452,
  "exp": 1511166652,
  "azp": "sAlZ...",
  "scope": "openid read:messages create:messages"
}
```

As you can see, the scopes are contained in the `scope` claim as a space delimited list. So the JWT above was issued with the `openid`, `read:messages` and `create:messages` scopes. If you now want to ensure that an access token passed to the API endpoint contains a particular scope, we can do the following:

First we can create a `HasScopeRequirement` with a `Scope` and `Issuer` property:

```csharp
public class HasScopeRequirement : IAuthorizationRequirement
{
    public string Issuer { get; }
    public string Scope { get; }

    public HasScopeRequirement(string scope, string issuer)
    {
        Scope = scope ?? throw new ArgumentNullException(nameof(scope));
        Issuer = issuer ?? throw new ArgumentNullException(nameof(issuer));
    }
}
```

Then create an `AuthorizationHandler` which will check whether the `scope` claim of the token contains the particular claim which is required, also ensuring that the `scope` claim was issued by the `Issuer` which was specified. In the case of an access token issued by Auth0, the Issuer will be the Auth0 tenant, for example `https://jerrie.auth0.com/` in the example specified above.

```csharp
public class HasScopeHandler : AuthorizationHandler<HasScopeRequirement>
{
    protected override Task HandleRequirementAsync(AuthorizationHandlerContext context, HasScopeRequirement requirement)
    {
        // If user does not have the scope claim, get out of here
        if (!context.User.HasClaim(c => c.Type == "scope" && c.Issuer == requirement.Issuer))
            return Task.CompletedTask;

        // Split the scopes string into an array
        var scopes = context.User.FindFirst(c => c.Type == "scope" && c.Issuer == requirement.Issuer).Value.Split(' ');

        // Succeed if the scope array contains the required scope
        if (scopes.Any(s => s == requirement.Scope))
            context.Succeed(requirement);

        return Task.CompletedTask;
    }
}
```

Next you will need to register a singleton for the `HasScopeHandler` class:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // ... some code omitted

    // register the scope authorization handler
    services.AddSingleton<IAuthorizationHandler, HasScopeHandler>();
}
```

Then go and add a policy **for each** claim you may want to check for in your API. For example if you want to authorize endpoints for the `read:messages` and `create:messages` scopes, you can create policies for those by calling the `AddPolicy` method:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // Add framework services.
    services.AddMvc();

    string domain = $"https://{Configuration["Auth0:Domain"]}/";
    services.AddAuthentication(options =>
    {
        options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
        options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;

    }).AddJwtBearer(options =>
    {
        options.Authority = domain;
        options.Audience = Configuration["Auth0:ApiIdentifier"];
    });
            
    services.AddAuthorization(options =>
    {
        options.AddPolicy("read:messages", policy => policy.Requirements.Add(new HasScopeRequirement("read:messages", domain)));
        options.AddPolicy("create:messages", policy => policy.Requirements.Add(new HasScopeRequirement("create:messages", domain)));
    });

    // register the scope authorization handler
    services.AddSingleton<IAuthorizationHandler, HasScopeHandler>();
}
```

Now, inside your API, you can simply specify that policy in the `Authorize` attribute:

```csharp
[Route("api/messages")]
public class MessagesController : Controller
{
    [Authorize("read:messages")]
    [HttpGet]
    public IActionResult GetAll()
    {
        // Code omitted...
    }

    [Authorize("create:messages")]
    [HttpPost]
    public IActionResult Create([FromBody] Message message)
    {
        // Code omitted...
    }
}
```

In the source sample above, the `GetAll` action will only be authorized if the access token passed in contains a `read:messages` scope, and the `Create` action will only be authorized if the access token contains a `create:messages` scope.

For more information you can also refer to [Custom policy-based authorization](https://docs.microsoft.com/en-us/aspnet/core/security/authorization/policies).

## Resolving policies dynamically

With that backgrounder out of the way, we can finally get to the purpose of this blog post, and that is to resolve the policies dynamically.

As you can probably imagine from the code above, you will need to call `AddPolicy` for each scope you want to check for. If you have many scopes you want to check for, then that list of policies can grow quite long.

As demonstrated by Dominick and Brock in the video at the beginning, you can resolve these dynamically by creating your own class which inherits from `DefaultAuthorizationPolicyProvider`. What this class will do is to check whether the policy request exist, and if it does it will return that policy.

However, if the policy does not exist, it will assume that we want to restrict access to a scope with that name, and so it will dynamically create an instance of `HasScopeRequirement`, and use the `policyName` parameter as the name of the scope which we want to check for:

```csharp
public class AuthorizationPolicyProvider : DefaultAuthorizationPolicyProvider
{
    private readonly AuthorizationOptions _options;
    private readonly IConfiguration _configuration;

    public AuthorizationPolicyProvider(IOptions<AuthorizationOptions> options, IConfiguration configuration) : base(options)
    {
        _options = options.Value;
        _configuration = configuration;
    }

    public override async Task<AuthorizationPolicy> GetPolicyAsync(string policyName)
    {
        // Check static policies first
        var policy = await base.GetPolicyAsync(policyName);

        if (policy == null)
        {
            policy = new AuthorizationPolicyBuilder()
                .AddRequirements(new HasScopeRequirement(policyName, $"https://{_configuration["Auth0:Domain"]}/"))
                .Build();

            // Add policy to the AuthorizationOptions, so we don't have to re-create it each time
            _options.AddPolicy(policyName, policy);
        }

        return policy;
    }
}
```

**Edit 1 July 2018**: I updated the code above to add the new policy to the `AuthorizationOptions` so that you do not have to create it each time. Thank you to Nick Flower who pointed this out to me. You can also [check out his blog post](https://0xnf.github.io/posts/oauthserver/15/#dynamically-handling-policies) about this.

You will also need to register the `AuthorizationPolicyProvider` as a singleton in `ConfigureServices`, as well as remove all the calls to `AddPolicy`, as the policies will now be resolved dynamically.

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // Add framework services.
    services.AddMvc();

    string domain = $"https://{Configuration["Auth0:Domain"]}/";
    services.AddAuthentication(options =>
    {
        options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
        options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;

    }).AddJwtBearer(options =>
    {
        options.Authority = domain;
        options.Audience = Configuration["Auth0:ApiIdentifier"];
    });
            
    services.AddAuthorization();

    // register the scope authorization handler
    services.AddSingleton<IAuthorizationPolicyProvider, AuthorizationPolicyProvider>();
    services.AddSingleton<IAuthorizationHandler, HasScopeHandler>();
}
```

With that small change in place the authorization policies will now be dynamically resolved at runtime, without you having to call `AddPolicy` for every scope you want to check for. 

Very neat little trick 😀
