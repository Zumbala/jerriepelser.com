---
date: 2015-11-10T00:00:00Z
description: |
  I demonstrate how you can allow users to save even more fine grained culture settings preferences against their profile.
tags:
- asp.net
- asp.net 5
- asp.net mvc
- localization
title: 'Allowing user to set culture settings in ASP.NET 5: Part 2'
url: /blog/allowing-user-to-set-culture-settings-aspnet5-part2/
---

## Introduction

In my [previous blog post](http://www.jerriepelser.com/blog/allowing-user-to-set-culture-settings-aspnet5-part1) I demonstrated how we can allow the user to save their culture preferences, and then apply the correct culture preferences once the user has signed in.

We allowed the user to set the Culture and the UI Culture, but I think in a real world scenario that is not very realistic. Typically we will want to give the user more fine-grained control to set their language, date format, time format, currency etc.

Before we get to that, let us first cover a couple of principles.

**Firstly**, let us talk about the difference between the Culture and the UI Culture.

The UI Culture is used to load resource files, so if you created resources for different languages, these will load depending on the value of the UI Culture - so the UI Culture determines the language our user interface is rendered in.

The Culture on the other hand is used to for formatting culture dependent values, such as date formats, time formats, the currency symbol, an so forth.

**The second thing** I want to mention is that in the previous example we loaded pre-defined culture settings. So when I create a new instance of CultureInfo class and pass in a culture name, such as

```csharp
var ci = new CultureInfo("af-ZA")
``` 

it will return a culture settings for Afrikaans (South Africa). It is important to note however that we do not need to create a `CultureInfo` instance with these pre-defined settings. We can override any of the settings for a `CultureInfo` settings if we want to.

So in my example, the default short date format for the culture `af-ZA` is `yyyy-mm-dd`, but I do not like it. I prefer the format of `dd/mm/yyyy`, so I can initialize a `CultureInfo` instance with the Afrikaans (South Africa) settings, and then override the date format to suit me:

```csharp
var ci = new CultureInfo("af-ZA")
ci.DateTimeFormat.ShortDatePattern = "dd/MM/yyyy";
```

So in understanding these two principles, we are going to extend the example from the [previous blog post](http://www.jerriepelser.com/blog/allowing-user-to-set-culture-settings-aspnet5-part1) to allow the user to select a language, and then also specify their preferred date, time and currency formats.

## Changing the database model

First thing to do is to change the database model to support the new culture specific settings we want to store per user. (I am leaving the previous `Culture` and `UICulture` properties from the previous blog post for now, otherwise I will get a bunch of compiler errors. We can remove them later):

```csharp
public class ApplicationUser : IdentityUser
{
    public string Culture { get; set; }
    public string UICulture { get; set; }

    public string CurrencySymbol { get; set; }
	[Required]
    public string Language { get; set; }
    public string LongDateFormat { get; set; }
    public string ShortDateFormat { get; set; }
}
```

I also want to set the default language to English, so I override the `OnModelCreating` method of my database context to make the language field required and also specify a default value of "en":

```csharp
public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
{
    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);

        builder.Entity<ApplicationUser>(typeBuilder =>
            {
                typeBuilder.Property<string>(u => u.Language)
                    .HasDefaultValue("en")
                    .IsRequired();
            });
    }
}
```

Like the last time I create a new migration with the `dnx ef migrations add` command and execute the migrations with `dnx ef database update`:

![](/assets/images/allowing-user-to-set-culture-settings-aspnet5-part2/execute-dnx-commands.png)

## Changing the user profile screen

I am also changing the screen which will allow the user to update their culture preferences. First up I change the `ConfigureCultureInfoViewModel` class with properties for the new fiels, and also the lookups allowing the user to select pre-defined values for the various culture settings:

``` csharp
public class ConfigureCultureInfoViewModel
{
    public List<SelectListItem> AvailableCurrencySymbols { get; }

    public List<SelectListItem> AvailableLanguages { get; }

    public List<SelectListItem> AvailableLongDateFormats { get; }

    public List<SelectListItem> AvailableShortDateFormats { get; }
    public string CurrencySymbol { get; set; }
    public string Language { get; set; }
    public string LongDateFormat { get; set; }
    public string ShortDateFormat { get; set; }

    public ConfigureCultureInfoViewModel()
    {
        AvailableCurrencySymbols = new List<SelectListItem>
        {
            // Source code omitted for brevity. 
        };

        AvailableLanguages = new List<SelectListItem>
        {
            // Source code omitted for brevity. 
        };

        AvailableLongDateFormats = new List<SelectListItem>
        {
            // Source code omitted for brevity. 
        };

        AvailableShortDateFormats = new List<SelectListItem>
        {
            // Source code omitted for brevity. 
        };
    }
}
``` 

I also update the Razor view:

``` html
@model ConfigureCultureInfoViewModel

@{
    ViewData["Title"] = "Configure your culture settings";
}

<h2>@ViewData["Title"].</h2>

<form asp-controller="Manage" asp-action="ConfigureCultureInfo" method="post" class="form-horizontal" role="form">
    <div class="form-group">
        <label asp-for="Language" class="col-sm-2 control-label"></label>
        <div class="col-sm-10">
            <select asp-for="Language" asp-items="Model.AvailableLanguages" class="form-control"></select>
        </div>
    </div>
    <div class="form-group">
        <label asp-for="ShortDateFormat" class="col-sm-2 control-label"></label>
        <div class="col-sm-10">
            <select asp-for="ShortDateFormat" asp-items="Model.AvailableShortDateFormats" class="form-control"></select>
        </div>
    </div>
    <div class="form-group">
        <label asp-for="LongDateFormat" class="col-sm-2 control-label"></label>
        <div class="col-sm-10">
            <select asp-for="LongDateFormat" asp-items="Model.AvailableLongDateFormats" class="form-control"></select>
        </div>
    </div>
    <div class="form-group">
        <label asp-for="CurrencySymbol" class="col-sm-2 control-label"></label>
        <div class="col-sm-10">
            <select asp-for="CurrencySymbol" asp-items="Model.AvailableCurrencySymbols" class="form-control"></select>
        </div>
    </div>
    <div class="form-group">
        <div class="col-md-offset-2 col-md-10">
            <button type="submit" class="btn btn-default">Save Settings</button>
        </div>
    </div>
</form>
``` 

And the `ManageController` class:

``` csharp
[HttpGet]
public async Task<IActionResult> ConfigureCultureInfo()
{
    var user = await _userManager.FindByIdAsync(User.GetUserId());

    var viewModel = new ConfigureCultureInfoViewModel
    {
        Language = user.Language,
        ShortDateFormat = user.ShortDateFormat,
        LongDateFormat = user.LongDateFormat,
        CurrencySymbol = user.CurrencySymbol
    };

    return View(viewModel);
}

[HttpPost]
public async Task<IActionResult> ConfigureCultureInfo(ConfigureCultureInfoViewModel viewModel)
{
    var user = await _userManager.FindByIdAsync(User.GetUserId());
    user.Language = viewModel.Language;
    user.ShortDateFormat = viewModel.ShortDateFormat;
    user.LongDateFormat = viewModel.LongDateFormat;
    user.CurrencySymbol = viewModel.CurrencySymbol;

    await _userManager.UpdateAsync(user);
    await _signInManager.SignInAsync(user, true); // Force the CreateUserPrincipalAsync method on our CustomSignInManager to be called again

    return RedirectToAction("Index");
}
``` 

## Updating the middleware extensions

In the previous blog post I created a custom `RequestCultureProvider` class to specify set the correct culture based on the user's settings. Also remember that I stored the Culture and UI Culture as claims in the authentication cookie. I am going to change that to store a JSON serialized instance of a new `UserCulturePreferences` class I will create:

``` csharp
public class UserCulturePreferences
{
    [JsonProperty("c")]
    public string CurrencySymbol { get; set; }
    [JsonProperty("l")]
    public string Language { get; set; }
    [JsonProperty("ld")]
    public string LongDateFormat { get; set; }
    [JsonProperty("sd")]
    public string ShortDateFormat { get; set; }
}
```

> **Please note** that because this JSON-serialized instance is stored in the cookie, you would want to keep it as small as possible, therefore I chose to serialize the property names to very short strings, such as "c", "l" etc. 
> 
> I still store the actual date formats as long strings however, which is not very efficient and adds to the size of the cookie. You may want to rather just store an integer representation of the various formats and store that inside the database (and subsequently the cookie) and convert it at runtime to a full format string. 

I also need to change the `CustomSignInManager` class to store the settings in the new object and then store a serialized representation in the claim:

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
            var userPreferences = new UserCulturePreferences
            {
                Language = user.Language,
                ShortDateFormat = user.ShortDateFormat,
                LongDateFormat = user.LongDateFormat,
                CurrencySymbol = user.CurrencySymbol
            };

            if (!string.IsNullOrEmpty(user.Culture))
                identity.AddClaim(new Claim("localizationapp:cultureprefs", JsonConvert.SerializeObject(userPreferences)));
        }

        return principal;
    }
}
```

Next up is the `PrincipalExtensions` class. I change that to deserialize the claim back to the `UserCulturePreferences` instance:

```csharp
public static class PrincipalExtensions
{
    public static UserCulturePreferences GetCulture(this ClaimsPrincipal principal)
    {
        if (principal == null)
            throw new ArgumentNullException(nameof(principal));

        var userPreferences = principal.FindFirstValue("localizationapp:cultureprefs");

        if (userPreferences == null)
            return null;

        return JsonConvert.DeserializeObject<UserCulturePreferences>(userPreferences);
    }
}
```

And finally I also change the `UserProfileRequestCultureProvider` to set the correct cultures:

``` csharp
public class UserProfileRequestCultureProvider : RequestCultureProvider
{
    public override Task<RequestCulture> DetermineRequestCulture(HttpContext httpContext)
    {
        if (httpContext == null)
            throw new ArgumentNullException(nameof(httpContext));

        if (!httpContext.User.Identity.IsAuthenticated)
            return Task.FromResult((RequestCulture)null);

        var culturePreferences = httpContext.User.GetCulture();
        if (culturePreferences == null)
            return Task.FromResult((RequestCulture)null);


        var uiCulture = new CultureInfo(culturePreferences.Language);
        var culture = new CultureInfo(culturePreferences.Language);

        if (!string.IsNullOrEmpty(culturePreferences.ShortDateFormat))
            culture.DateTimeFormat.ShortDatePattern = culturePreferences.ShortDateFormat;
        if (!string.IsNullOrEmpty(culturePreferences.LongDateFormat))
            culture.DateTimeFormat.LongDatePattern = culturePreferences.LongDateFormat;
        if (!string.IsNullOrEmpty(culturePreferences.CurrencySymbol))
            culture.NumberFormat.CurrencySymbol = culturePreferences.CurrencySymbol;

        var requestCulture = new RequestCulture(culture, uiCulture);
        return Task.FromResult(requestCulture);
    }
}
```

Note that I set both the culture and the UI culture to whatever the language is that the user chose, and then override the culture object's properties with the correct date formats and currency symbol. This will ensure that long dates will contain the correct localized month and week names in the language the user selected.
 
## Adding localized strings

In order to demonstrate the effect of all the localized settings, I will add some resource files with localized strings. I add a resource file with a resource string named "WelcomeText":  

![](/assets/images/allowing-user-to-set-culture-settings-aspnet5-part2/resource-file.png)

I then add 3 resource files for the different languages my application supports, each with the translation of the localized text: 

![](/assets/images/allowing-user-to-set-culture-settings-aspnet5-part2/solution-explorer-resources.png)

> Please also see the section "Issues with current Tooling" below. Also, I used Google Translate for the Spanish translation, so it is probably not very correct :(

I also need to make a change to my `ConfigureServices` method to support localization for MVC. To do that I call the `AddViewLocalization()` extension method after adding all the MVC services, and pass the path where the resources are located:

``` csharp
// Add MVC services to the services container.
services.AddMvc()
    .AddViewLocalization(options => options.ResourcesPath = "Resources");
```

This is my final `ConfigureServices` method:

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

    // Add MVC services to the services container.
    services.AddMvc()
        .AddViewLocalization(options => options.ResourcesPath = "Resources");

    // Register application services.
    services.AddTransient<IEmailSender, AuthMessageSender>();
    services.AddTransient<ISmsSender, AuthMessageSender>();
}
```

## Update Home Page to display settings

The last thing I want to do is to update the home page to display a few localized values, so we can be sure everything is working fine:

``` html
@using System.Threading.Tasks
@using LocalizationByUser.Resources
@using Microsoft.AspNet.Mvc.Localization
@{
    ViewData["Title"] = "Home Page";
}
@inject IHtmlLocalizer<AppResources> SR

<div class="container">
    <div class="row">
        <div class="col-md-12">
            <div>Text in user's language: @SR["WelcomeText"]</div>
            <div>Sample Long Date: @DateTime.Now.ToString("D")</div>
            <div>Sample Short Date: @DateTime.Now.ToString("d")</div>
            <div>Sample currency: @(1000.ToString("C"))</div>
        </div>
    </div>
</div>
```

I have injected an instance of `IHtmlLocalizer<AppResources>` which will give me access to the "WelcomeText" resource string which is located in the `AppResources.resx` file, and then I just display the localized version of that resource string. I also display the current date in the correct long and short date formats, as well as display the value of "1000" formatted as a currency.

So browsing to the user culture settings page again, I configure the values for the various settings:

![](/assets/images/allowing-user-to-set-culture-settings-aspnet5-part2/culture-settings-page.png)

When I save it and browse back to the home page you can see that the correct culture settings are used:

![](/assets/images/allowing-user-to-set-culture-settings-aspnet5-part2/home-page.png)

## Issues with current Tooling

There are currently a few issues with the tooling which makes working with localization a bit of a challenge. 

If you run the application from Visual Studio the correct localized text is not displayed, so you need to run it from the command line. There is an issue logged for it:
[https://github.com/aspnet/Localization/issues/115](https://github.com/aspnet/Localization/issues/115)

Also, there is currently no way to add a .RESX file to an ASP.NET 5 project, so you need to create one in an ASP.NET 4.6 project and then copy it over:
[https://github.com/aspnet/Localization/issues/117](https://github.com/aspnet/Localization/issues/117)

When you edit the above mentioned file, it will always reset the visibility of the class in `.designer.cs` file to `internal`. You will need to go and edit it manually to change it back to `public`.
 
## Conclusion

In this blog post I extended the sample from the previous blog post to allow user's to specify much more fine-grained culture settings by allowing them to specify the language, date formats and currency symbol. This example is not perfect, but it is a good start for you should you want to employ a similar method.
