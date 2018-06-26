---
title: "Authenticate with OAuth 2.0 in ASP.NET Core 2.0"
description: |
  An introduction to the generic OAuth 2.0 authentication handler in ASP.NET Core 2.0 which allow you to authenticate users using any OAuth 2.0 provider.
date: "2018-01-04"
tags:
- asp.net
- asp.net core
- oauth
- github
url: /blog/authenticate-oauth-aspnet-core-2
---

Almost 2 years ago I wrote a blog post about [using the generic OAuth provider in ASP.NET Core](/blog/introduction-to-aspnet5-generic-oauth-provider/). A lot has changed since then, so I thought it might be a good time to revisit this.

For this blog post, we will create a simple website which allows users to sign in with their GitHub credentials. I will also demonstrate how we can store the access token received from GitHub to make GitHub API calls using [Octokit](https://github.com/octokit/octokit.net).

## The OAuth flow

Before we get going, I would like to go through the OAuth 2 flow quickly so you can understand how things fit together.

Regarding terminology, I will be referring to Consumers and Service Providers. A Consumer is an application that will be requesting an OAuth token, so, for example, our ASP.NET Core application. The Service Provider is the application or service which authorizes the user and issues the token. 

In this blog post I will demonstrate how to configure authentication with GitHub, so for the sake of this blog post think of GitHub as the Service Provider.

Before anything can be done, you will need to register an application with the Service Provider. Typically you will need to specify a name for the application and a **redirect URI**. Once an application is registered, the Service Provider will provide a **client ID** and a **client secret** which is used during the authentication and token request process.

As for the actual OAuth 2 flow, it looks as follows: 

![](/assets/images/2015-04-14-introduction-to-aspnet5-generic-oauth-provider/oauth-flow.png)

1. The Consumer makes a request to the Service Provider **authorization endpoint** to authorize the user.
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

The Client ID and secret are obtained when registering your application on the Service Provider's website. Some Service Providers may call them something else, like Facebook, for example, will refer to them as the App ID and App Secret. The Authorization and Token endpoints can typically be obtained from the developer documentation of the Service Provider.

The redirect URI is something which you can specify when configuring the OAuth authentication on the Consumer. This needs to match up with whatever you specify when creating the application on the Service Provider. This will normally be referred to as either a **redirect URL** or **callback URL** or some sort of variation of that.

Using our example of GitHub, this information can be found in the OAuth section of the API documentation at https://developer.github.com/v3/oauth/

You will see they list the Authorization endpoint as

``` text
https://github.com/login/oauth/authorize
```

and the Token endpoint as

``` text
https://github.com/login/oauth/access_token
```

The Authorization endpoint will usually have the path `/authorize`, `/authenticate`, `/auth` or something similar. The Token endpoint will typically have the path `/access_token` or `/token`, so look out for those.

## Register a GitHub application

To start off with, you will need to register an application by going to the [GitHub Developer Settings](https://github.com/settings/developers):

![GitHub OAuth applications](/assets/images/2018-01-04-authenticate-oauth-aspnet-core-2/oauth-apps.png)

Click on the button to **Register a new application**, and complete the information for your application. Specify `http://localhost:5000/signin-github` as the value for the **Authorization callback URL** field:

![Register new OAuth application](/assets/images/2018-01-04-authenticate-oauth-aspnet-core-2/register-oauth-app.png)

Once done you can click the **Register application** button. Take note of the values for **Client ID** and **Client Secret**, as you will need those shortly when registering the OAuth middleware.

![Your new OAuth application](/assets/images/2018-01-04-authenticate-oauth-aspnet-core-2/oauth-app-settings.png)

## Create the ASP.NET Core application

In Visual Studio, select File > New Project and select the **ASP.NET Core Web Application** template:

![Create a new project](/assets/images/2018-01-04-authenticate-oauth-aspnet-core-2/new-project.png)

In the next dialog, select the **Web Application** template, which will create a Razor Pages application for you. Also, make sure that the **Authentication** setting is set to **No Authentication**:

![Select ASP.NET project type](/assets/images/2018-01-04-authenticate-oauth-aspnet-core-2/new-aspnet-app.png)

If you want to create an MVC application, then go ahead and select the **Web Application (Model-View-Controller)** template. You should still be able to follow along as the important stuff is done in the `Startup` file which is the same for Razor Pages and MVC applications.

Once the project has been created you can go to your project Properties, and on the **Debug** page ensure you set the **App URL** to `http://localhost:5000/`. Alternatively, you need to change the application registration in GitHub to use the same port as your application.

![Configure the App URL](/assets/images/2018-01-04-authenticate-oauth-aspnet-core-2/app-properties.png)

If you are using the command line utilities, you can run the following command to create the same project:

```text
dotnet new razor -n AspNetCoreGitHubAuth
```

## Register the authentication middleware

You will need to register the Cookie and OAuth authentication services and also add the authentication middleware. First off, update the `ConfigureServices` method of your `Startup` class to register the required authentication services:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc();

    services.AddAuthentication(options =>
        {
            options.DefaultAuthenticateScheme = CookieAuthenticationDefaults.AuthenticationScheme;
            options.DefaultSignInScheme = CookieAuthenticationDefaults.AuthenticationScheme;
            options.DefaultChallengeScheme = "GitHub";
        })
        .AddCookie()
        .AddOAuth("GitHub", options =>
        {
            options.ClientId = Configuration["GitHub:ClientId"];
            options.ClientSecret = Configuration["GitHub:ClientSecret"];
            options.CallbackPath = new PathString("/signin-github");

            options.AuthorizationEndpoint = "https://github.com/login/oauth/authorize";
            options.TokenEndpoint = "https://github.com/login/oauth/access_token";
            options.UserInformationEndpoint = "https://api.github.com/user";

            options.ClaimActions.MapJsonKey(ClaimTypes.NameIdentifier, "id");
            options.ClaimActions.MapJsonKey(ClaimTypes.Name, "name");
            options.ClaimActions.MapJsonKey("urn:github:login", "login");
            options.ClaimActions.MapJsonKey("urn:github:url", "html_url");
            options.ClaimActions.MapJsonKey("urn:github:avatar", "avatar_url");

            options.Events = new OAuthEvents
            {
                OnCreatingTicket = async context =>
                {
                    var request = new HttpRequestMessage(HttpMethod.Get, context.Options.UserInformationEndpoint);
                    request.Headers.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
                    request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", context.AccessToken);

                    var response = await context.Backchannel.SendAsync(request, HttpCompletionOption.ResponseHeadersRead, context.HttpContext.RequestAborted);
                    response.EnsureSuccessStatusCode();

                    var user = JObject.Parse(await response.Content.ReadAsStringAsync());

                    context.RunClaimActions(user);
                }
            };
        });
}
```

The call to `AddAuthentication` registers the authentication services. It specified the `DefaultAuthenticateScheme` and the `DefaultSignInScheme` as the cookies, and the `DefaultChallengeScheme` as **GitHub**.  What this means is that when ASP.NET check to see whether a user is authenticated, it will use the cookie authentication handler. 

When we call `ChallengeAsync()` to log a user in, the **GitHub** authentication scheme (in other words the OAuth authentication handler) will be challenged. Finally, once the user is authenticated, their authentication information will be saved in a cookie, since that is the `DefaultSignInScheme`.

If this does not make sense to you, I **strongly** suggest that your read [ASP.NET Core 2.0 Authentication and Authorization System Demystified](https://digitalmccullough.com/posts/aspnetcore-auth-system-demystified.html) which explains all of this in much more detail.

Next up we register the cookie authentication handler with a call to `AddCookie()`.

Then we register the OAuth authentication handler by calling the `AddOAuth()` method and setting the `authenticationScheme` parameter as **GitHub** (the same we specified for the `DefaultChallengeScheme` earlier).

For the OAuth authentication handler, you need to specify the `ClientId`, `ClientSecret` and `CallbackPath`. The `CallbackPath` is the path where the identity provider will call back to after the user has authenticated (i.e. the callback URL we specified when we registered our application in GitHub).

We also specify the `AuthorizationEndpoint`, `TokenEndpoint` and `UserInformationEndpoint`. The latter is not called by the OAuth authentication handler itself, but instead is something we need to call to obtain more information about the user (name, email address etc.)

This is what is happening in the `OnCreatingTicket` event in the code above. We call the `UserInformationEndpoint` and receive a JSON response from GitHub. That response looks similar to the following:

```json
{
  "login": "octocat",
  "id": 1,
  "avatar_url": "https://github.com/images/error/octocat_happy.gif",
  "gravatar_id": "",
  "url": "https://api.github.com/users/octocat",
  "html_url": "https://github.com/octocat",
  "followers_url": "https://api.github.com/users/octocat/followers",
  "following_url": "https://api.github.com/users/octocat/following{/other_user}",
  "gists_url": "https://api.github.com/users/octocat/gists{/gist_id}",
  "starred_url": "https://api.github.com/users/octocat/starred{/owner}{/repo}",
  "subscriptions_url": "https://api.github.com/users/octocat/subscriptions",
  "organizations_url": "https://api.github.com/users/octocat/orgs",
  "repos_url": "https://api.github.com/users/octocat/repos",
  "events_url": "https://api.github.com/users/octocat/events{/privacy}",
  "received_events_url": "https://api.github.com/users/octocat/received_events",
  "type": "User",
  "site_admin": false,
  "name": "monalisa octocat",
  "company": "GitHub",
  "blog": "https://github.com/blog",
  "location": "San Francisco",
  "email": "octocat@github.com",
  "hireable": false,
  "bio": "There once was...",
  "public_repos": 2,
  "public_gists": 1,
  "followers": 20,
  "following": 0,
  "created_at": "2008-01-14T04:33:35Z",
  "updated_at": "2008-01-14T04:33:35Z"
}
```

In our case we want to use the `id` field for the `NameIdentifier` claim, `name` for the `Name` and also some other claims containing the user's GitHub `login` etc. We inform the OAuth authentication handler of this by first specifying the following `ClaimActions` in the authentication handler registration:

```csharp
options.ClaimActions.MapJsonKey(ClaimTypes.NameIdentifier, "id");
options.ClaimActions.MapJsonKey(ClaimTypes.Name, "name");
options.ClaimActions.MapJsonKey("urn:github:login", "login");
options.ClaimActions.MapJsonKey("urn:github:url", "html_url");
options.ClaimActions.MapJsonKey("urn:github:avatar", "avatar_url");
```

And then subsequently calling `RunClaimActions`, passing along the user object received from GitHub. This will ensure that the correct claims are created on our `ClaimsIdentity`.

The other part which we need to do is to register the authentication middleware. You can do that by calling `UseAuthentication()` in your `Configure` method:

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
        app.UseBrowserLink();
    }
    else
    {
        app.UseExceptionHandler("/Error");
    }

    app.UseStaticFiles();

    app.UseAuthentication();

    app.UseMvc(routes =>
    {
        routes.MapRoute(
            name: "default",
            template: "{controller}/{action=Index}/{id?}");
    });
}
```

## Update the application settings

You will have noticed that the OAuth authentication configuration referenced the `GitHub:ClientId` and `GitHub:ClientSecret` configuration settings. You will need to update your `appsettings.json` file to add these settings. Specify the values for your own GitHub application which you registered earlier.

```json
{
  "Logging": {
    "IncludeScopes": false,
    "LogLevel": {
      "Default": "Warning"
    }
  },
  "GitHub": {
    "ClientId": "your client id",
    "ClientSecret": "your client secret"
  }
}
```

> I store these values directly in `appsettings.json` for demonstration purposes only, but this is not good practice. Please refer to [Safe storage of app secrets during development in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets) for some better ways to handle this.

## Challenging the authentication handler

We need a way to challenge the authentication handler. For this I will create a normal MVC Controller called `AccountController` and specify a `Login` action which will return a `ChallengeResult` and therefore challenge the OAuth authentication handler (because we specified **GitHub** as the `DefaultChallengeScheme`):

```csharp
[Route("[controller]/[action]")]
public class AccountController : Controller
{
    [HttpGet]
    public IActionResult Login(string returnUrl = "/")
    {
        return Challenge(new AuthenticationProperties() { RedirectUri = returnUrl });
    }
}
```

## Call the Login action

Next, we can add a big button on the Home page which allows the user to log in with their GitHub credentials. This button will simply call the `Login` action on the `Controller` class. Update the HTML for your `/Pages/Index.cshtml` file as follows:

```html
@page
@model IndexModel
@{
    ViewData["Title"] = "Home page";
}

<div class="row">
    <div class="col-md-12">
        <a asp-action="Login" asp-controller="Account" class="btn btn-default">Log In with GitHub</a>
    </div>
</div>
```

## Display user information

One more thing we can do is to display the user's information. For this we can update the code for the `IndexModel` class in `/Pages/Index.cshtml.cs` as follows:

```csharp
public class IndexModel : PageModel
{
    public string GitHubAvatar { get; set; }

    public string GitHubLogin { get; set; }

    public string GitHubName { get; set; }

    public string GitHubUrl { get; set; }

    public void OnGet()
    {
        if (User.Identity.IsAuthenticated)
        {
            GitHubName = User.FindFirst(c => c.Type == ClaimTypes.Name)?.Value;
            GitHubLogin = User.FindFirst(c => c.Type == "urn:github:login")?.Value;
            GitHubUrl = User.FindFirst(c => c.Type == "urn:github:url")?.Value;
            GitHubAvatar = User.FindFirst(c => c.Type == "urn:github:avatar")?.Value;
        }
    }
}
```
Note that we retrieve the claims stored for the user, and store the values for those claims in properties on our `IndexModel ` class. This will allow use to use these values from the Razor page.

You can update the HTML code for the Razor page (located at `/Pages/Index.cshtml`) as follows:

```html
@page
@model IndexModel
@{
    ViewData["Title"] = "Home page";
}

<div class="row">
    <div class="col-md-12">
        @if (!User.Identity.IsAuthenticated)
        {
            <a asp-action="Login" asp-controller="Account" class="btn btn-default">Log In with GitHub</a>
        }
        else
        {
            <div class="row">
                <div class="col-md-2">
                    <img src="@Model.GitHubAvatar" alt="" class="img-thumbnail" />
                </div>
                <div class="col-md-10">
                    <h4>@Model.GitHubName</h4>
                    <p><i class="fab fa-github"></i><a href="@Model.GitHubUrl">@Model.GitHubLogin</a></p>
                </div>
            </div>
        }
    </div>
</div>
```

## Trying it out

Let's try this out by running the application. You will be presented with the home page which displays a Log In button:

![Log in button](/assets/images/2018-01-04-authenticate-oauth-aspnet-core-2/login-button.png)

Clicking on the button will redirect you to GitHub where you will be asked whether you want to Authorize the application to access your information:

![Authorize the application](/assets/images/2018-01-04-authenticate-oauth-aspnet-core-2/authorize-application.png)

If you click on the **Authorize** button you will be redirected back you the application where you can see the user's profile information:

![The user's profile](/assets/images/2018-01-04-authenticate-oauth-aspnet-core-2/user-profile.png)

## Retrieving the user's repositories

Let's say that we want to display the list of repositories for the user on the page as well. To do this, we need to call the [`/user/repos`](https://developer.github.com/v3/repos/#list-your-repositories) endpoint of the GitHub API, passing along the `access_token` which we received from GitHub.

Let's install [Octokit](https://github.com/octokit/octokit.net), which is a C# wrapper around the GitHub API:

```text
Install-Package Octokit
```

We will need to save the tokens received from GitHub. We can do that by setting the `SaveTokens` property when we configure the OAuth authentication handler:

```csharp
services.AddAuthentication(options =>
    {
        options.DefaultAuthenticateScheme = CookieAuthenticationDefaults.AuthenticationScheme;
        options.DefaultSignInScheme = CookieAuthenticationDefaults.AuthenticationScheme;
        options.DefaultChallengeScheme = "GitHub";
    })
    .AddCookie()
    .AddOAuth("GitHub", options =>
    {
        options.ClientId = Configuration["GitHub:ClientId"];
        options.ClientSecret = Configuration["GitHub:ClientSecret"];
        options.CallbackPath = new PathString("/signin-github");

        options.AuthorizationEndpoint = "https://github.com/login/oauth/authorize";
        options.TokenEndpoint = "https://github.com/login/oauth/access_token";
        options.UserInformationEndpoint = "https://api.github.com/user";

        options.SaveTokens = true;

        // some code omitted for brevity

        options.Events = new OAuthEvents
        {
            OnCreatingTicket = async context =>
            {
                // some code omitted for brevity
            }
        };
    });
```                

Then in the `IndexModel` class, we can retrieve the access token by calling the `GetTokenAsync()` method, and subsequently using that to initialize the `GitHubClient` class and retrieve the repositories for the current user. We store the repositories in a new `Repositories` property so that we can use it on the Razor page.

Also note that since we are now using `async` methods, I have changed the `OnGet()` method to `OnGetAsync()`.

```csharp
public class IndexModel : PageModel
{
    public string GitHubAvatar { get; set; }

    public string GitHubLogin { get; set; }

    public string GitHubName { get; set; }

    public string GitHubUrl { get; set; }

    public IReadOnlyList<Repository> Repositories { get; set; }

    public async Task OnGetAsync()
    {
        if (User.Identity.IsAuthenticated)
        {
            GitHubName = User.FindFirst(c => c.Type == ClaimTypes.Name)?.Value;
            GitHubLogin = User.FindFirst(c => c.Type == "urn:github:login")?.Value;
            GitHubUrl = User.FindFirst(c => c.Type == "urn:github:url")?.Value;
            GitHubAvatar = User.FindFirst(c => c.Type == "urn:github:avatar")?.Value;

            string accessToken = await HttpContext.GetTokenAsync("access_token");

            var github = new GitHubClient(new ProductHeaderValue("AspNetCoreGitHubAuth"), 
                new InMemoryCredentialStore(new Credentials(accessToken)));
            Repositories = await github.Repository.GetAllForCurrent();

        }
    }
}
```

Also, update the Razor page to display the list of repositories:

```html
@page
@model IndexModel
@{
    ViewData["Title"] = "Home page";
}

<div class="row">
    <div class="col-md-12">
        @if (!User.Identity.IsAuthenticated)
        {
            <a asp-action="Login" asp-controller="Account" class="btn btn-default">Log In with GitHub</a>
        }
        else
        {
            <div class="row">
                <div class="col-md-2">
                    <img src="@Model.GitHubAvatar" alt="" class="img-thumbnail" />
                </div>
                <div class="col-md-10">
                    <h4>@Model.GitHubName</h4>
                    <p><i class="fab fa-github"></i><a href="@Model.GitHubUrl">@Model.GitHubLogin</a></p>
                    <h4>Repositories</h4>
                    <ul>
                        @foreach (var repo in Model.Repositories)
                        {
                            <li><a href="@repo.HtmlUrl">@repo.FullName</a></li>
                        }
                    </ul>
                </div>
            </div>
        }
    </div>
</div>
```

Now when we run the application and log in again, you will notice that the user's repositories are displayed:

![The user's profile with their repositories](/assets/images/2018-01-04-authenticate-oauth-aspnet-core-2/user-profile-with-github-repositories.png)

## Source code

Source code for this sample is available on GitHub at [https://github.com/jerriepelser-blog/AspnetCoreGitHubAuth](https://github.com/jerriepelser-blog/AspnetCoreGitHubAuth)