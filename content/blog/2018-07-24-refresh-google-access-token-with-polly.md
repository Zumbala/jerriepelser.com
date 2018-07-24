---
title: "Refresh a Google Access Token with Polly"
description:
  Google access tokens are short-lived, but you can configure a Polly policy to refresh the access token when expired.
tags:
- asp.net core
- .net core
- polly
- httpclient
---

## Introduction

I have an application which communicates with Google Drive via the [Goole Drive REST API](https://developers.google.com/drive/api/v3/about-sdk). I am not using the official Google SDK since my use case is confined to a few API endpoints, and I also want to use the new `IHttpClientFactory` [that was released in ASP.NET Core 2.1](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/http-requests?view=aspnetcore-2.1).

The one problem I ran into was the fact that the access tokens issued by Google are short-lived. When authorizing the user in Google, my application requests a refresh token and stores this along with the user's access token. I have then configured [Polly](https://github.com/App-vNext/Polly) to refresh the access token if it has expired and retry the request with the newly issued access token.

## Creating a Typed client

The `IHttpClientFactory` in ASP.NET Core allows you to [create Typed clients](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/http-requests?view=aspnetcore-2.1#typed-clients). I used this technique to define the following `GoogleDriveApi` class:

```csharp
public class GoogleDriveApi
{
    private readonly HttpClient _client;

    public GoogleDriveApi(HttpClient client)
    {
        _client = client;

        _client.BaseAddress = new Uri("https://www.googleapis.com/");
    }
}
```

I subsequently registered in my Typed client with the ASP.NET Core Dependency Injection:

```csharp
public class Startup
{
    // Some code omitted for brevity...

    public void ConfigureServices(IServiceCollection services)
    {
        //...

        services
            .AddHttpClient<GoogleDriveApi>();

        services.AddMvc()
            .SetCompatibilityVersion(CompatibilityVersion.Version_2_1);
    }
}
```

For this blog post, let's confine ourselves to a single endpoint on the Google Drive API, in particular, the endpoint which will [list a user's files](https://developers.google.com/drive/api/v3/reference/files/list). I created the following method on my `GoogleDriveApi` class which will call the endpoint:

```csharp
public class GoogleDriveApi
{
    // Code omitted for brevity...

    public async Task<ListFilesResponse> ListFiles(string accessToken)
    {
        var requestMessage = new HttpRequestMessage(HttpMethod.Get, "/drive/v3/files");
        requestMessage.Headers.Add("Authorization", $"Bearer {accessToken}");

        var response = await _client.SendAsync(requestMessage);

        response.EnsureSuccessStatusCode();

        return await response.Content.ReadAsAsync<ListFilesResponse>();
    }
}
```

This works fine until the access token expires, when the endpoint will return an HTTP Status 401 (Unauthorized). Let's look at how we can configure a policy with Polly which will refresh the access token, and retry the request with the new access token.

## Configure a Polly policy

Ensure that you have installed Polly:

```bash
dotnet add package Polly
```

First, let's update the `ListFiles` method. The first thing I did is to expand the list of parameters to also include the `refreshToken`. I also added a `tokenRefreshed` delegate which will allow this method to call back to my application code when a new access token was requested. This will allow my application to update the user's access token in the database for subsequent requests.

Also, notice that I pass the access token and refresh tokens in the Polly [Context data](https://github.com/App-vNext/Polly/wiki/Keys-and-Context-Data). Inside the `ExecuteAsync` method, I use the access token from the context data as the bearer token.

```csharp
public class GoogleDriveApi
{
    // Code omitted for brevity...

    public async Task<ListFilesResponse> ListFiles(string accessToken, string refreshToken, Func<string, Task> tokenRefreshed)
    {
        var policy = CreateTokenRefreshPolicy(tokenRefreshed);

        var response = await policy.ExecuteAsync(context =>
        {
            var requestMessage = new HttpRequestMessage(HttpMethod.Get, "/drive/v3/files");
            requestMessage.Headers.Add("Authorization", $"Bearer {context["access_token"]}");

            return _client.SendAsync(requestMessage);
        }, new Dictionary<string, object>
        {
            {"access_token", accessToken},
            {"refresh_token", refreshToken}
        });

        response.EnsureSuccessStatusCode();

        return await response.Content.ReadAsAsync<ListFilesResponse>();
    }
}
```

Now, let's look at that `CreateTokenRefreshPolicy` method. First, I create a new policy that handles HTTP Status code 401 (Unauthorized). The policy will retry the request one time, and on retry, it will check the Polly Context data for a refresh token. If a refresh token exists, it calls the `RefreshAccessToken` method (see code below) to refresh the access token using that refresh token.

If we have obtained a new access token, it will call that `tokenRefreshed` delegate that will allow me to update the user's existing access token in the database with the newly issued one. Finally, I update the access token in the context data with the new access token, so that the second attempt will use this new access token.

```csharp
public class GoogleDriveApi
{
    // Code omitted for brevity...

    private RetryPolicy<HttpResponseMessage> CreateTokenRefreshPolicy(Func<string, Task> tokenRefreshed)
    {
        var policy = Policy
            .HandleResult<HttpResponseMessage>(message => message.StatusCode == HttpStatusCode.Unauthorized)
            .RetryAsync(1, async (result, retryCount, context) =>
            {
                if (context.ContainsKey("refresh_token"))
                {
                    var newAccessToken = await RefreshAccessToken(context["refresh_token"].ToString());
                    if (newAccessToken != null)
                    {
                        await tokenRefreshed(newAccessToken);

                        context["access_token"] = newAccessToken;
                    }
                }
            });

        return policy;
    }
}
```

The last piece of the puzzle is the `RefreshAccessToken` method. This method calls the Google token endpoint to exchange the refresh token for a new access token:

```csharp
public class GoogleDriveApi
{
    // Code omitted for brevity...

    private async Task<string> RefreshAccessToken(string refreshToken)
    {
        var refreshMessage = new HttpRequestMessage(HttpMethod.Post, "/oauth2/v4/token")
        {
            Content = new FormUrlEncodedContent(new KeyValuePair<string, string>[]
            {
                new KeyValuePair<string, string>("client_id", _configuration["Authentication:Google:ClientId"]),
                new KeyValuePair<string, string>("client_secret", _configuration["Authentication:Google:ClientSecret"]),
                new KeyValuePair<string, string>("refresh_token", refreshToken),
                new KeyValuePair<string, string>("grant_type", "refresh_token")
            })
        };

        var response = await _client.SendAsync(refreshMessage);

        if (response.IsSuccessStatusCode)
        {
            var tokenResponse = await response.Content.ReadAsAsync<TokenResponse>();

            return tokenResponse.AccessToken;
        }

        // return null if we cannot request a new token
        return null;
    }
}
```

You will notice that this method accesses the application's configuration to get the Client ID and Client Secret for my Google application. This will also get injected into my `GoogleDriveApi` constructor at runtime:


```csharp
public class GoogleDriveApi
{
    private readonly HttpClient _client;
    private readonly IConfiguration _configuration;

    public GoogleDriveApi(HttpClient client, IConfiguration configuration)
    {
        _client = client;
        _configuration = configuration;

        _client.BaseAddress = new Uri("https://www.googleapis.com/");
    }

    // Code omitted for brevity...
}
```

## Calling the Google Drive API

Using this class is quite easy. At runtime an instance gets injected into my controller's constructor, which I then store in a field named `_googleDriveApi`. Then, when I want to call it, I have a bit of code which retrieves the user's access token and refresh token from the database. I call the `ListFiles` method using those values, and I also pass the delegate which called when the token is refreshed, which will update the access token for the user in the database.

```csharp
// Retrieve access token and refresh token from database
string accessToken = "...";
string refreshToken = "...";

// Get the list of files from Google Drive
var filesResponse = await _googleDriveApi.ListFiles(accessToken, refreshToken, async token =>
{
    // Update access token in the database
    // ...
});
```

## In action

Here's what it looks like in action. In the Fiddler screenshot below you will notice that there are two sets of requests. In the first set, the user goes to the `/app/documents` URL of my application. This controller action will make a call to the Google Drive API to list the user's documents.

The first request to the `/drive/v3/files` endpoint returns an HTTP Status 401 (Unauthorized). A request is then made to the `/oauth2/v4/token` endpoint to refresh the access token, and a second request is made to the `/drive/v3/files` endpoint which returns an HTTP Status 200 (OK) since the newly issued access token was used.

In the second set of requests, the user once again goes to the `/app/documents` URL. Once again, a request is made to the `/drive/v3/files` endpoint which returns an HTTP Status 200 (OK) the first time, since the access token that was just issued is still valid.

![Requests in Fiddler](/images/blog/2018-07-24-refresh-google-access-token-with-polly/fiddler-output.png)

## Conclusion

Using Polly you can fairly easily implement patterns such as refreshing an access token and retrying a request with the newly issued access token.