---
date: 2017-04-10T00:00:00Z
description: |
  Shows how you can manually validate a JSON Web Token using .NET
tags:
- .net core
- .net
- jwt
title: Manually validating a JWT using .NET
url: /blog/manually-validating-rs256-jwt-dotnet/
---

[JSON Web Tokens](https://jwt.io/) are commonly used to authorize request made to an API. For this purpose ASP.NET (both OWIN and Core) has middleware which allows you to easily authorize any request by ensuring the token being passed to the API is valid. 

But what if you want to manually validate a token?

At [Auth0](https://auth0.com/) we allow signing of tokens using either a symmetric algorithm (HS256), or an asymmetric algorithm (RS256). HS256 tokens are signed and verified using a simple secret, where as RS256 use a private and public key for signing and verifying the token signatures.

See this blog post by my colleague Shawn Meyer on [Navigating RS256 and JWKS](https://auth0.com/blog/navigating-rs256-and-jwks/) for more information.

Well back to the question of validating a token, and in this case specifically a token signed using the RS256 algorithm.

The source code for the ASP.NET Core JWT middleware is [available on GitHub](https://github.com/aspnet/Security/tree/dev/src/Microsoft.AspNetCore.Authentication.JwtBearer) and browsing through that gives some clues as to how you can achieve this in a non-ASP.NET application.

First up ensure that you have the following NuGet packages installed:

```text
Install-Package System.IdentityModel.Tokens.Jwt
Install-Package Microsoft.IdentityModel.Protocols.OpenIdConnect
```

Since this mostly geared towards using this technique with Auth0, I am declaring some variables containing my Auth0 Domain and Audience (which with typically your Auth0 API Identifier). Depending on your scenario you may need to adapt this to suit your needs.

```csharp
const string auth0Domain = "https://jerrie.auth0.com/"; // Your Auth0 domain
const string auth0Audience = "https://rs256.test.api"; // Your API Identifier
```

The first thing is to download the OIDC Configuration from the [OpenID Connect Discovery endpoint](https://openid.net/specs/openid-connect-discovery-1_0.html). This will contain (among other things) the JSON Web Key Set containing the public key(s) that can be used to verify the token signature.

```csharp
IConfigurationManager<OpenIdConnectConfiguration> configurationManager = new ConfigurationManager<OpenIdConnectConfiguration>($"{auth0Domain}.well-known/openid-configuration", new OpenIdConnectConfigurationRetriever());
OpenIdConnectConfiguration openIdConfig = await configurationManager.GetConfigurationAsync(CancellationToken.None);
```

Next up we need to configure the token validation parameters. I specify the issuer and audience(s) and also tell it to use the signing keys - i.e. the public key(s) - which were downloaded above.

```csharp
TokenValidationParameters validationParameters =
    new TokenValidationParameters
    {
        ValidIssuer = auth0Domain,
        ValidAudiences = new[] { auth0Audience },
        IssuerSigningKeys = openIdConfig.SigningKeys
    };
```

With that in place, all you need to do is validate the token:

```csharp
SecurityToken validatedToken;
JwtSecurityTokenHandler handler = new JwtSecurityTokenHandler();
var user = handler.ValidateToken("eyJhbGciOi.....", validationParameters, out validatedToken);
```

`ValidateToken` will return a [`ClaimsPrincipal`](https://msdn.microsoft.com/en-us/library/system.security.claims.claimsprincipal(v=vs.110).aspx) which will contain all the claims from the JSON Web Token.

So for example, to get the user's ID, we can query the `NameIdentifier` claim:

```csharp
Console.WriteLine($"Token is validated. User Id {user.Claims.FirstOrDefault(c => c.Type == ClaimTypes.NameIdentifier)?.Value}");
```

You can find the sample application I did for our Auth0 samples at [https://github.com/auth0-samples/auth0-dotnet-validate-jwt/tree/master/IdentityModel-RS256](https://github.com/auth0-samples/auth0-dotnet-validate-jwt/tree/master/IdentityModel-RS256)