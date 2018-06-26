---
date: 2015-11-03T00:00:00Z
description: |
  I demonstrate how you can allow users to save their culture settings preferences against their profile.
tags:
- asp.net
- asp.net 5
- asp.net mvc
- localization
title: 'Allowing user to set culture settings in ASP.NET 5: Part 1'
url: /blog/allowing-user-to-set-culture-settings-aspnet5-part1/
---

In a [previous blog post](http://www.jerriepelser.com/blog/how-aspnet5-determines-culture-info-for-localization) I demonstrated how ASP.NET 5 determines which Culture settings to use when serving a request. That is fine for many cases, but in some instances you want to allow users to save their culture settings preferences in their user profile.

This is Part 1 in a 2 part series when I will demonstrate how you can do this. In this blog post I will show you how we can allow a user to pick a specific culture and then save that against their user profile. In the next blog post I will demonstrate how we can allow the user to store the culture settings at an even more fine-grained level.

## Getting started

To start off with I created a simple ASP.NET 5 application using Visual Studio, and ensure I have set Authentication to "Individual User accounts" when I did that. Next up I headed over to the `Startup.cs` class and under the `Configure` method I enabled request localization:

``` csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    loggerFactory.MinimumLevel = LogLevel.Information;
    loggerFactory.AddConsole();
    loggerFactory.AddDebug();

    // Configure the Localization middleware
    app.UseRequestLocalization();
	
	// Rest of method omitted for brevity
}
```

I also want to change my user model to allow users to store their Culture and UI Culture preferences, so let's open `Models\ApplicationUser.cs` and add two fields to store the culture information:

``` csharp
public class ApplicationUser : IdentityUser
{
    public string Culture { get; set; }

    public string UICulture { get; set; }
}
```

Next up from the command line, ensure you are in the folder of the Web application, and execute the commmand

``` text 
dnx ef migrations add AddUserCultureSettings
```

This will add a new migration to add the necessary columns to the database for the new fields we added to the `ApplicationUser` class. 

We want to execute the migrations, so run the command

``` text
dnx ef database update
```

This will ensure that all outstanding migrations are applied to the database.

![](/assets/images/allowing-user-to-set-culture-settings-aspnet5-part1/execute-dnx-commands.png)

## Allowing user to change settings

To allow the user to configure their culture settings I will add a new View Model to the `ViewModels\Manage` folder called `ConfigureCultureInfoViewModel`:

``` csharp
public class ConfigureCultureInfoViewModel
{
    public string Culture { get; set; }

    public string UICulture { get; set; } 

    public List<SelectListItem> AvailableCultures { get; }

    public ConfigureCultureInfoViewModel()
    {
        AvailableCultures = new List<SelectListItem>
        {
            new SelectListItem
            {
                Value = "",
                Text = "-- Not specified --"
            },
            new SelectListItem
            {
                Value = "af",
                Text = "Afrikaans"
            },
            new SelectListItem
            {
                Value = "ca",
                Text = "Catalan"
            },
            new SelectListItem
            {
                Value = "en",
                Text = "English"
            },
            new SelectListItem
            {
                Value = "es",
                Text = "Spanish"
            }
        };
    }
}
```

And also create the relevant controller actions to allow the user to change these settings:

```csharp
[HttpGet]
public async Task<IActionResult> ConfigureCultureInfo()
{
    var user = await _userManager.FindByIdAsync(User.GetUserId());

    var viewModel = new ConfigureCultureInfoViewModel
    {
        Culture = user.Culture,
        UICulture = user.UICulture
    };

    return View(viewModel);
}

[HttpPost]
public async Task<IActionResult> ConfigureCultureInfo(ConfigureCultureInfoViewModel viewModel)
{
    var user = await _userManager.FindByIdAsync(User.GetUserId());
    user.Culture = viewModel.Culture;
    user.UICulture = viewModel.UICulture;

    await _userManager.UpdateAsync(user);

    return RedirectToAction("Index");
}
```

As well as the Razor view:
 
``` html
@model ConfigureCultureInfoViewModel

@{
    ViewData["Title"] = "Configure your culture settings";
}

<h2>@ViewData["Title"].</h2>

<form asp-controller="Manage" asp-action="ConfigureCultureInfo" method="post" class="form-horizontal" role="form">
    <div class="form-group">
        <label asp-for="Culture" class="col-sm-2 control-label"></label>
        <div class="col-sm-10">
            <select asp-for="Culture" asp-items="Model.AvailableCultures" class="form-control"></select>
        </div>
    </div>
    <div class="form-group">
        <label asp-for="UICulture" class="col-sm-2 control-label"></label>
        <div class="col-sm-10">
            <select asp-for="UICulture" asp-items="Model.AvailableCultures" class="form-control"></select>
        </div>
    </div>
    <div class="form-group">
        <div class="col-md-offset-2 col-md-10">
            <button type="submit" class="btn btn-default">Save Settings</button>
        </div>
    </div>
</form>
```

And lastly I update the existing Manage Index view (`Views\Manage\Index.cshtml`) to put a link to the new page to manage their Culture settings:

```html
@model IndexViewModel
@{
    ViewData["Title"] = "Manage your account";
}

<h2>@ViewData["Title"].</h2>
<p class="text-success">@ViewData["StatusMessage"]</p>
<div>
    <h4>Change your account settings</h4>
    <hr />
    <dl class="dl-horizontal">
		<!-- Parts of view omitted for brevity -->

        <dt>Culture Settings:</dt>
        <dd>
            <a asp-controller="Manage" asp-action="ConfigureCultureInfo">Change</a>
        </dd>
    </dl>
</div>
```

## Storing the culture settings in a claim

We are going to have to extend the Localization middleware to set the culture settings for the request based on the user's settings. We will however need to get the culture info from the User table in the database, but we do not want to make a query to the database everytime we do this.

The way we can prevent this is to store the Culture settings in a claim when the user signs in, so we have to create a custom sign in manager and override the `CreateUserPrincipalAsync` method to add the new claims we need:

``` csharp
public class CustomSignInManager : SignInManager<ApplicationUser>
{
    public CustomSignInManager(UserManager<ApplicationUser> userManager, IHttpContextAccessor contextAccessor, IUserClaimsPrincipalFactory<ApplicationUser> claimsFactory,
        IOptions<IdentityOptions> optionsAccessor, ILogger<SignInManager<ApplicationUser>> logger)
        : base(userManager, contextAccessor, claimsFactory, optionsAccessor, logger)
    {
    }

    public override async Task<ClaimsPrincipal> CreateUserPrincipalAsync(ApplicationUser user)
    {
        var principal = await base.CreateUserPrincipalAsync(user);

        var identity = principal.Identity as ClaimsIdentity;
        if (identity != null)
        {
            if (!string.IsNullOrEmpty(user.Culture))
                identity.AddClaim(new Claim("localizationapp:culture", user.Culture));

            if (!string.IsNullOrEmpty(user.UICulture))
                identity.AddClaim(new Claim("localizationapp:uiculture", user.UICulture));
        }

        return principal;
    }
}
```

In the `Startup.cs` class, in the `ConfigureServices` method, I register the new `CustomSignInManager` class:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // Add Entity Framework services to the services container.
    services.AddEntityFramework()
        .AddSqlServer()
        .AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(Configuration["Data:DefaultConnection:ConnectionString"]));

    // Add Identity services to the services container.
    services.AddIdentity<ApplicationUser, IdentityRole>(options =>
                                                        {
                                                            options.Password.RequireDigit = false;
                                                            options.Password.RequireLowercase = false;
                                                            options.Password.RequireNonLetterOrDigit = false;
                                                            options.Password.RequireUppercase = false;
                                                            options.Password.RequiredLength = 6;
                                                        })
        .AddEntityFrameworkStores<ApplicationDbContext>()
        .AddDefaultTokenProviders();

    services.AddScoped<SignInManager<ApplicationUser>, CustomSignInManager>();

    // Rest of method omitted for brevity
}
```

Finally I also created two helper methods on the `ClaimsPrincipal` class to allow me to more easily get the Culture and UI Culture from the claims:


``` csharp
public static class PrincipalExtensions
{
    public static string GetCulture(this ClaimsPrincipal principal)
    {
        if (principal == null)
            throw new ArgumentNullException(nameof(principal));

        return principal.FindFirstValue("localizationapp:culture");
    }

    public static string GetUICulture(this ClaimsPrincipal principal)
    {
        if (principal == null)
            throw new ArgumentNullException(nameof(principal));

        return principal.FindFirstValue("localizationapp:uiculture");
    }
}
```

One problem however is that `CustomSignInManager.CreateUserPrincipalAsync()` only going to be called when the user signs in. So once the user is signed in and they update the settings, the claims will not be updated. For now the only way I have figured out to do this is to call `SignInManager.SignInAsync()` again, so I updated the controller action when the user saves the Culture settings to do that:

```csharp
[HttpPost]
public async Task<IActionResult> ConfigureCultureInfo(ConfigureCultureInfoViewModel viewModel)
{
    var user = await _userManager.FindByIdAsync(User.GetUserId());
    user.Culture = viewModel.Culture;
    user.UICulture = viewModel.UICulture;

    await _userManager.UpdateAsync(user);
    await _signInManager.SignInAsync(user, true); // Force the CreateUserPrincipalAsync method on our CustomSignInManager to be called again 

    return RedirectToAction("Index");
}
```

## Create a custom Request Culture Provider

Phew. I hope you're still keeping up. The next and final step is to create our own Request Culture Provider. Remember when I spoke about [how ASP.NET 5 determines the Culture for a request](http://www.jerriepelser.com/blog/how-aspnet5-determines-culture-info-for-localization)?. In that blog post I mentioned the list of Request Culture Providers which are defined in the `RequestLocalizationOptions` class. 

What we need to do now is to go and write our own class and then register it along with the other providers. To create the Request Culture Provider I borrow the code from Microsoft's [QueryStringRequestCultureProvider](https://github.com/aspnet/Localization/blob/dev/src/Microsoft.AspNet.Localization/QueryStringRequestCultureProvider.cs) class and adapt it for my own purposes:

```csharp
public class UserProfileRequestCultureProvider : RequestCultureProvider
{
    public override Task<RequestCulture> DetermineRequestCulture(HttpContext httpContext)
    {
        if (httpContext == null)
            throw new ArgumentNullException(nameof(httpContext));

        if (!httpContext.User.Identity.IsAuthenticated)
            return Task.FromResult((RequestCulture)null);

        string userCulture = null;
        string userUICulture = null;

        string cultureClaim = httpContext.User.GetCulture();
        if (!string.IsNullOrWhiteSpace(cultureClaim))
        {
            userCulture = cultureClaim;
        }

        string uicultureClaim = httpContext.User.GetUICulture();
        if (!string.IsNullOrWhiteSpace(uicultureClaim))
        {
            userUICulture = uicultureClaim;
        }

        if (userCulture == null && userUICulture == null)
        {
            // No values specified for either so no match
            return Task.FromResult((RequestCulture)null);
        }

        if (userCulture != null && userUICulture == null)
        {
            // Value for culture but not for UI culture so default to culture value for both
            userUICulture = userCulture;
        }

        if (userCulture == null && userUICulture != null)
        {
            // Value for UI culture but not for culture so default to UI culture value for both
            userCulture = userUICulture;
        }

        var requestCulture = new RequestCulture(new CultureInfo(userCulture), new CultureInfo(userUICulture));

        return Task.FromResult(requestCulture);
    }
}
```

So I get the values for the culture and the UI culture from the claims. First I check whether the request is authenticated, and if not I return null. If neither value is set then I also return null so one of the other providers can provide the correct information. Also, if only one of the values is set, then I will default the other to that value.

Finally I need to register the new provider, so I will need to update the call to `app.UseRequestLocalization()` in my `Configure` method. The thing is however that the Localization middleware is now dependent on the Authentication middleware having executed, because otherwise I will not be able to access the claims I required to determine the user's preferred culture.

So I need to move my call to `UseRequestLocalization` below the call to `UseIdentity()` which registers the cookie authentication middleware. Also, I need to register the new `UserProfileRequestCultureProvider` as one of the possible culture providers, so I create an instance of `RequestLocalizationOptions` and then add the instance of `UserProfileRequestCultureProvider` to the beginning of the list of `RequestCultureProviders`, so it takes the highest priority:

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    // Code omitted for brevity

    // Add cookie-based authentication to the request pipeline.
    app.UseIdentity();

    // Configure the Localization middleware after ASP.NET Identity has been configured
    var requestLocalizationOptions = new RequestLocalizationOptions();
    requestLocalizationOptions.RequestCultureProviders.Insert(0, new UserProfileRequestCultureProvider());
    app.UseRequestLocalization(requestLocalizationOptions);

    // Code omitted for brevity
}
```

> Note that because I am moving the Localization middleware to execute later in the request pipeline, all middleware registered before that will not be able to take advantage of Localization. In my case this is not an issue, as it is only really the error pages which will now not be able to take advantage of Localization, and I am not too concerned about that. I want to you however to be aware of this, so make sure this does not affect any other aspect of your application in an adverse manner.  

## Testing it out
	
To test it out I registered a user and signed in. If I go to the how page I display the Culture and UI Culture used for the request:

![](/assets/images/allowing-user-to-set-culture-settings-aspnet5-part1/demo1.png)

Currently it displays "English (United Kingdom)" because that is what my browser is set to.

I click on the "Hello jerrie@jerriepelser.com" on the top right which navigates me to the User management section:

![](/assets/images/allowing-user-to-set-culture-settings-aspnet5-part1/demo2.png)

I click on the Change link next to "Culture Settings", which navigates me to the Culture Settings configuration screen. I update the culture settings to "Afrikaans" and click save:

![](/assets/images/allowing-user-to-set-culture-settings-aspnet5-part1/demo3.png)

Now when I navigate back to the home page you can see that the Culture settings are updated to Afrikaans:

![](/assets/images/allowing-user-to-set-culture-settings-aspnet5-part1/demo4.png)

## Conclusion

In this blog post I demonstrated how we can allow the user to specify their own culture settings. In the next post I will demonstrate how we can allow the user even more flexibility by allowing them to override their date and time formats.

For non-signed in users, or users who have not specified specific culture settings, the same rules apply as I described in [How ASP.NET Determines the Culture Settings for Localization](http://www.jerriepelser.com/blog/how-aspnet5-determines-culture-info-for-localization). When a user is signed in and they have chosen to specify custom culture settings, those will take highest priority over all the other methods which determine the culture settings.