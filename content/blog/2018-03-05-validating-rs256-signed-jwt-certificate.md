---
title: "Manually supplying certificate when validating RS256 signed JWT with OWIN"
description: |
  Shows how you can validate RS256 signed tokens in the OWIN JWT middleware using an offline certificate.
date: 2018-03-05
tags:
- asp.net
- owin
- jwt
url: /blog/validating-rs256-signed-jwt-certificate
---

In the [Auth0 OWIN Quickstart sample](https://auth0.com/docs/quickstart/backend/webapi-owin) we use a little helper class which will download the [JSON Web Key Set (JWKS)](https://auth0.com/docs/jwks) for the Auth0 tenant and use that in the `IssuerSigningKeyResolver` delegate to verify the JWT's signature.

That code looks as follows:

```csharp
public void Configuration(IAppBuilder app)
{
    var domain = $"https://{ConfigurationManager.AppSettings["Auth0Domain"]}/";
    var apiIdentifier = ConfigurationManager.AppSettings["Auth0ApiIdentifier"];

    var keyResolver = new OpenIdConnectSigningKeyResolver(domain);
    app.UseJwtBearerAuthentication(
        new JwtBearerAuthenticationOptions
        {
            AuthenticationMode = AuthenticationMode.Active,
            TokenValidationParameters = new TokenValidationParameters()
            {
                ValidAudience = apiIdentifier,
                ValidIssuer = domain,
                IssuerSigningKeyResolver = (t, st, i, p) => keyResolver.GetSigningKey(i)
            }
        });

    // Configure Web API
    WebApiConfig.Configure(app);
}
```

Turns out there are cases when people, for whatever reasons, want to manually download and specify the certificate which is used to verify the signature. Let's look at two possible ways you can do this with Auth0.

## Use the .cer file

The first way to go about this is to download the `.cer` file for the Auth0 tenant. This can be done from the `https://AUTH0_TENANT_DOMAIN/cer` endpoint. Go ahead and save that file in your project folder as `auth0.cer`:

![](/assets/images/2018-03-05-validating-rs256-signed-jwt-certificate/cer-file.png)

Verify that the **Build Action** is set to **Content** to ensure that file gets copied along when you publish the website:

![](/assets/images/2018-03-05-validating-rs256-signed-jwt-certificate/file-properties.png)

In the `Configuration` method of your OWIN `Startup` class you can then simply load this file from disk and create a new `X509Certificate2` instance. Then, in the `IssuerSigningKeyResolver` delegate, you can return the certificate:

```csharp
public void Configuration(IAppBuilder app)
{
    var domain = $"https://{ConfigurationManager.AppSettings["Auth0Domain"]}/";
    var apiIdentifier = ConfigurationManager.AppSettings["Auth0ApiIdentifier"];

    string certificatePath = HostingEnvironment.MapPath("~/auth0.cer");
    var certificate = new X509Certificate2(certificatePath);

    app.UseJwtBearerAuthentication(
        new JwtBearerAuthenticationOptions
        {
            AuthenticationMode = AuthenticationMode.Active,
            TokenValidationParameters = new TokenValidationParameters()
            {
                ValidAudience = apiIdentifier,
                ValidIssuer = domain,
                IssuerSigningKeyResolver = (t, st, i, p) => new X509SecurityKey(certificate),
            }
        });

    // Configure Web API
    WebApiConfig.Configure(app);
}
```

## Use the Signing Certificate value

The second way is to use the Signing Certificate value which contains the Base64-encoded certificate. Go to the [Clients section of the Auth0 Dashboard](https://manage.auth0.com/#/clients) and go to the **Settings** for any of your Clients. Click **Show Advanced Settings** and go to the **Certificate** tab. Copy the value of the **Signing Certificate** without the "-----BEGIN CERTIFICATE-----" and "-----END CERTIFICATE-----" delimiters:

![](/assets/images/2018-03-05-validating-rs256-signed-jwt-certificate/signing-certificate.png)

You can now use that to create a `X509Certificate2` and then subsequently use the certificate in the `IssuerSigningKeyResolver` delegate:

```csharp
public void Configuration(IAppBuilder app)
{
    var domain = $"https://{ConfigurationManager.AppSettings["Auth0Domain"]}/";
    var apiIdentifier = ConfigurationManager.AppSettings["Auth0ApiIdentifier"];

    string publicKeyBase64 = @"MIIC6DCCAdCgAw...1WouHHA6uUQ==";
    var certificate = new X509Certificate2(Convert.FromBase64String(publicKeyBase64));

    app.UseJwtBearerAuthentication(
        new JwtBearerAuthenticationOptions
        {
            AuthenticationMode = AuthenticationMode.Active,
            TokenValidationParameters = new TokenValidationParameters()
            {
                ValidAudience = apiIdentifier,
                ValidIssuer = domain,
                IssuerSigningKeyResolver = (t, st, i, p) => new X509SecurityKey(certificate),
            }
        });

    // Configure Web API
    WebApiConfig.Configure(app);
}
```
