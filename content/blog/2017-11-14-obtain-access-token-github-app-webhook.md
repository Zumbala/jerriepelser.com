---
title: "Obtaining an Access Token from a GitHub Application Webhook"
description: |
  Shows how to generate an access token to call the GitHub API when creating a Webhook for a GitHub application.
date: 2017-11-14
tags:
- github
- octokit
- webhook
- azure
- azure functions
url: /blog/obtain-access-token-github-app-webhook
---

When developing a [Webhook](https://developer.github.com/webhooks/) for a [GitHub application](https://developer.github.com/apps/), you may want to make calls to the [GitHub API](https://developer.github.com/v3/). In order to do this you will need to have a Bearer token to authorize the calls.

In a typical scenario, you would obtain a token as part of the user authorization flow, but when developing a webhook for a GitHub application things work a bit differently as there is no interactive user involved.

The process to obtain a token is described in [authentication options for GitHub Apps](https://developer.github.com/apps/building-integrations/setting-up-and-registering-github-apps/about-authentication-options-for-github-apps/). Sadly, the [GitHub API Client Library for .NET](https://github.com/octokit/octokit.net) does not have support for this, and I could also not find samples on how to do it.

So this blog post demonstrates this process. 

In my case I developed a GitHub webhook using [Azure Functions](https://azure.microsoft.com/en-us/services/functions/), but Azure Functions are not required.

## Authenticating as a GitHub App

As described in [About authentication options for GitHub Apps](https://developer.github.com/apps/building-integrations/setting-up-and-registering-github-apps/about-authentication-options-for-github-apps/), the first thing you will need to do is to authenticate as the GitHub app.

Ensure that you have [generated a private key](https://developer.github.com/apps/building-integrations/setting-up-and-registering-github-apps/registering-github-apps/#generating-a-private-key) which you will use to sign the JSON Web Token you generate. You can download the private key as a PEM file, the content of which looks similar to this:

```text
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAxOZJnjb9TDU5A752knBWYkKUTUbp/dS0xCyiYLo27rBr4nPw
AGtz37pSEvVbZIOd+YeXfSsyFITJ8tFHF4f+gLoZFPUDRpQ9jFEnaMk+k1Hpa++M
c54fIKkJJ7EXR5sgk4Hw8oyafJe7BWxMpjlv7nWv19l9JFGdEc9UTjuJJwmXB1sx
98r5dyO3PbetnXB0ERgzi1Dj23IwoGgblhEX48EYw5rmfdYAY9gUBxq5etuR93iG
kdHD8bFtZpkdK4m5+qpF3jqZC6G9S+KWKBT8GMpUumeaGWSHnJkRBFgt0+QVo/ac
3fg8r6WI41KXh1s0tnd1uy5CAA1KSWtIz2wczQIDAQABAoIBAGQNeR8yZigRa3Vh
lTL4bozjtCRIgqJ/MZP18lUgisVjVI7vxucPswPe09HaZ6R91Qp1Qh9SkdInX89J
VMy6SDL1XcVuc0msDlJ88n+ms2t81hYI1d/yW3LnMkW/EUtLopQSFXPKyr3BBEHP
NGvLB/uuRbpzQ8nuxtfD+N0AF8ODkdTDt/qONbKYgQAsDH+L2u0I9foJPxOpgAB8
vldOs3WR/g/TjMeecFZM7k1YP7396lFrrCwz+jcvjK8B2yy2JU3aVnSc1YftuQlg
QQrzWXPwAam63aoUdV3CM5lmNlI0rdG6GR/AY8x2cCge4X/K9PO1jaxLZ838wMnA
Udi6gukCgYEA4TrBhVZyUV3ycBaOAYfW3lWK22oK6FBYGIpVNOi8zdQNyEZp/2Hh
nO8Np5gC1K/MRANB5gKsSHMCvp9JkvNh3TAzpaBHiKokjiUlzuPrDssmEdXIcOP9
l/6ZKHAL8PvNLyhvW9bl25MKmNB8qEZex2zGUuiV8rn4hvif6Yy7fC8CgYEA38y2
HIIOz6qCoJS6eqC8a3YfOYMDxYqTI8VaK7FNoFNGfzKwfZ24lQxrUAUVDnTD/4DU
BMFjk5vB2w19rWBZiYXZlsOW9FzaBeiZbeaTk7y56V6hGlGg7sUZpnji+JAdLEOY
wkwTeXFkvPMAqcGTD2AmbrZMNJKwDeiX3uwBi8MCgYEAwi69IC962ioMVpUrk6hB
GWlZEXjotIhom6fQdJBgD3lO9Ke4xYTnVj1IYYEOl76FdvH0A9G0JbZ63ExKy5Zo
fqILNTXt1qcSYzcl1KzuQM/nH6NYb/Qw9LV96L5JnVgK/Ch5cg3iGSA6F6xQfWhG
WNuwVVTPyviJjByZCmvaTOMCgYAwt2UlC4Kh3eIVhoqZSwf49HuTj1tw0tt4u+sG
+8+ECNydr7EPhxe5Cj0J7DFLNku8AJUme9prNUqbnnIvHXwgHYzh7q2bf+LNynYr
oDebv0dQ75JuIjq//xO4UzRFG0neatCD6v+KPg7JurP4P/T9Mdbr3q7z+sJsJKd6
lTE49QKBgCEUyysuBpVTTIfuCfQrSLU0RsKOHpplIVShpRejauMaIZrAPj4ScE12
MVhIXnF+ypGqk+nNVIv6q13HMj2/E2OOaxkbqKB0TWbFAVyLjcl5iIqwgAYUUEbl
1saHiJvqYP8f8UAoKzM6XiH8R82PrGxoEOOOCDuUYVJ2KmrZkcnA
-----END RSA PRIVATE KEY-----
```

The first thing you will need to do is to convert your PEM file to XML format. Copy the contents of the PEM file, then head over to the [RSA Key Converter](https://superdry.apphb.com/tools/online-rsa-key-converter) website. Paste the contents in the textbox under the **PEM to XML** heading and click **Convert**

![Convert PEM to XML](/assets/images/2017-11-14-obtain-access-token-github-app-webhook/pem-to-xml.png)

This will generate XML similar to the following:

```xml
<RSAKeyValue>
    <Modulus>xOZJnjb9TDU5A752knBWYkKUTUbp/dS0xCyiYLo27rBr4nPwAGtz37pSEvVbZIOd+YeXfSsyFITJ8tFHF4f+gLoZFPUDRpQ9jFEnaMk+k1Hpa++Mc54fIKkJJ7EXR5sgk4Hw8oyafJe7BWxMpjlv7nWv19l9JFGdEc9UTjuJJwmXB1sx98r5dyO3PbetnXB0ERgzi1Dj23IwoGgblhEX48EYw5rmfdYAY9gUBxq5etuR93iGkdHD8bFtZpkdK4m5+qpF3jqZC6G9S+KWKBT8GMpUumeaGWSHnJkRBFgt0+QVo/ac3fg8r6WI41KXh1s0tnd1uy5CAA1KSWtIz2wczQ==</Modulus>
    <Exponent>AQAB</Exponent>
    <P>4TrBhVZyUV3ycBaOAYfW3lWK22oK6FBYGIpVNOi8zdQNyEZp/2HhnO8Np5gC1K/MRANB5gKsSHMCvp9JkvNh3TAzpaBHiKokjiUlzuPrDssmEdXIcOP9l/6ZKHAL8PvNLyhvW9bl25MKmNB8qEZex2zGUuiV8rn4hvif6Yy7fC8=</P>
    <Q>38y2HIIOz6qCoJS6eqC8a3YfOYMDxYqTI8VaK7FNoFNGfzKwfZ24lQxrUAUVDnTD/4DUBMFjk5vB2w19rWBZiYXZlsOW9FzaBeiZbeaTk7y56V6hGlGg7sUZpnji+JAdLEOYwkwTeXFkvPMAqcGTD2AmbrZMNJKwDeiX3uwBi8M=</Q>
    <DP>wi69IC962ioMVpUrk6hBGWlZEXjotIhom6fQdJBgD3lO9Ke4xYTnVj1IYYEOl76FdvH0A9G0JbZ63ExKy5ZofqILNTXt1qcSYzcl1KzuQM/nH6NYb/Qw9LV96L5JnVgK/Ch5cg3iGSA6F6xQfWhGWNuwVVTPyviJjByZCmvaTOM=</DP>
    <DQ>MLdlJQuCod3iFYaKmUsH+PR7k49bcNLbeLvrBvvPhAjcna+xD4cXuQo9CewxSzZLvACVJnvaazVKm55yLx18IB2M4e6tm3/izcp2K6A3m79HUO+SbiI6v/8TuFM0RRtJ3mrQg+r/ij4Oybqz+D/0/THW696u8/rCbCSnepUxOPU=</DQ>
    <InverseQ>IRTLKy4GlVNMh+4J9CtItTRGwo4emmUhVKGlF6Nq4xohmsA+PhJwTXYxWEhecX7KkaqT6c1Ui/qrXccyPb8TY45rGRuooHRNZsUBXIuNyXmIirCABhRQRuXWxoeIm+pg/x/xQCgrMzpeIfxHzY+sbGgQ444IO5RhUnYqatmRycA=</InverseQ>
    <D>ZA15HzJmKBFrdWGVMvhujOO0JEiCon8xk/XyVSCKxWNUju/G5w+zA97T0dpnpH3VCnVCH1KR0idfz0lUzLpIMvVdxW5zSawOUnzyf6aza3zWFgjV3/JbcucyRb8RS0uilBIVc8rKvcEEQc80a8sH+65FunNDye7G18P43QAXw4OR1MO3+o41spiBACwMf4va7Qj1+gk/E6mAAHy+V06zdZH+D9OMx55wVkzuTVg/vf3qUWusLDP6Ny+MrwHbLLYlTdpWdJzVh+25CWBBCvNZc/ABqbrdqhR1XcIzmWY2UjSt0boZH8BjzHZwKB7hf8r087WNrEtnzfzAycBR2LqC6Q==</D>
</RSAKeyValue>
```

Next, you will also need the Identifier of your Github application. You can find this under the **General** settings for your app, under the **About** heading:

![Copy GitHub App Identifier](/assets/images/2017-11-14-obtain-access-token-github-app-webhook/app-identifier.png)

We will use the `System.IdentityModel.Tokens.Jwt` NuGet package to generate the token, so go ahead and install it:

```text
Install-Package System.IdentityModel.Tokens.Jwt
```

Now you can use the XML you previously generated from the PEM file, along with the GitHub app identifier to generate the token. First you will use the `RSACryptoServiceProvider.FromXmlString` method to load the private key and create an `RsaSecurityKey` instance.

Then you can use the `JwtSecurityTokenHandler` class to create the JSON Web Token.

```csharp
using System.Security.Cryptography;
using Microsoft.IdentityModel.Tokens;
using System.IdentityModel.Tokens.Jwt;

private static string GenerateAppToken()
{
    // Values below are hardcoded in this sample snippet, but your should store it under the App Settings for your Azure function
    // and then retrieve it using Environment.GetEnvironmentVariable("SETTING_NAME", EnvironmentVariableTarget.Process);
    string privateKey = "<RSAKeyValue>...</RSAKeyValue>"; // The value you generated using the PEM to XML tool
    string gitHubAppIdentifier = "1234"; // Your GitHub Application Identifier

    // Load key
    var rsaProvider = new RSACryptoServiceProvider();
    rsaProvider.FromXmlString(privateKey);
    var key = new RsaSecurityKey(rsaProvider);

    // Create token using the JwtSecurityTokenHandler
    JwtSecurityTokenHandler handler = new JwtSecurityTokenHandler();
    var token = handler.CreateJwtSecurityToken(gitHubAppIdentifier, null, null, null, DateTime.Now.AddMinutes(1),
        DateTime.Now, new SigningCredentials(key, SecurityAlgorithms.RsaSha256));
    
    return token.RawData;
}
```

## Authenticating as an installation

The next step is to authenticate as the app installation. For this you will need the installation ID, which you can obtain from the payload of your webhook. In the case of GitHub applications, the webhook payload will contain an `installation` section with an `id` attribute.

Here is a sample pull request event payload: 

```json
{
  "action": "opened",
  "number": 1,
  "pull_request": {
      ...
  },
  "repository": {
      ...
  },
  "sender": {
      ...
  },
  "installation": {
    "id": 5678
  }
}
```

So you will need to extract that value from the payload and then use it to generate the installation token. In my case below I am passing it in as a parameter. Also note that the App token which was generated above is passed as a `Bearer` token in the `Authorization` header to authorize the request.

Here is the full code to generate an installation token:

```csharp
private static async Task<string> GenerateInstallationToken(string installationId)
{
    var httpClient = new HttpClient();
    var requestMessage = new HttpRequestMessage(HttpMethod.Post, $"https://api.github.com/installations/{installationId}/access_tokens");
    requestMessage.Headers.Add("User-Agent", "Static-Scheduler");
    requestMessage.Headers.Add("Authorization", $"Bearer {GenerateAppToken()}");
    requestMessage.Headers.Add("Accept", "application/vnd.github.machine-man-preview+json");
    var response = await httpClient.SendAsync(requestMessage);

    var content = await response.Content.ReadAsStringAsync();
    var token = JsonConvert.DeserializeObject<dynamic>(content);

    return token.token;
}
```

## Putting it together

We can now put it all together and use it inside an Azure Function. Our function also uses [Octokit](https://github.com/octokit/octokit.net) to determine whether the user opening a pull request is a collaborator on the repository. So go ahead and install Octokit.

```text
Install-Package Octokit
```

And this is what the sample Azure function looks like:

```csharp
public static async Task<HttpResponseMessage> Run([HttpTrigger(WebHookType = "github")] HttpRequestMessage req,
    TraceWriter log)
{
    string gitHubEvent = req.Headers.GetValues("X-GitHub-Event").FirstOrDefault();
    if (gitHubEvent == "pull_request")
    {
        //Get request body
        dynamic data = await req.Content.ReadAsAsync<object>();

        string userName = data.pull_request.user.login;
        long repositoryId = data.repository.id;
        string installationId = data.installation.id;

        // We will need to make some GitHub API calls, so let's generate a token for the Application installation
        string token = await GenerateInstallationToken(installationId);

        // Use Octokit to determine whether user is a collaborator on the repo
        var github = new GitHubClient(new ProductHeaderValue("Static-Scheduler"), new InMemoryCredentialStore(new Credentials(token)));
        bool isCollaborator = await github.Repository.Collaborator.IsCollaborator(repositoryId, userName);

        // Do some other processing...

        return req.CreateResponse(HttpStatusCode.Accepted, "response goes here...");
    }

    // Tell GitHub that we're happy
    return req.CreateResponse(HttpStatusCode.Accepted, $"Event {gitHubEvent} ignored");
}
```

Notice that I checked the `X-GitHub-Event` header to ensure I am only processing pull request events. Then I simply retrieve the installation ID from the payload, along with the repository ID and the username.

I use the installation ID to generate a token, and then use that token to make calls using the Octokit library.

You will need to enure that your GitHub app has the [correct permissions](https://developer.github.com/apps/building-integrations/setting-up-and-registering-github-apps/about-permissions-for-github-apps/) to call whichever endpoints you require.

**Big thanks to Piotr Gankiewicz** for [this blog post](http://piotrgankiewicz.com/2017/07/24/jwt-rsa-hmac-asp-net-core/) which showed me how to use the PEM file together with the .NET JWT classes to generate a JSON Web Token.

**Please note** that there is no error handling happening in my sample code. If you want to use this in production, you will need to add exception handling where appropriate 😉