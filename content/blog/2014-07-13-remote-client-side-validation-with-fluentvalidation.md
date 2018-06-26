---
date: 2014-07-13T00:00:00Z
description: |
  Extending your use of the FluentValidation library by integrating it with the remote validation in jQuery Validation.
tags:
- aspnet mvc
- fluent validation
- mvc
title: Remote Client Side Validation with FluentValidation
url: /blog/remote-client-side-validation-with-fluentvalidation/
---

My go-to library for model validation in .NET is [Fluent Validation](http://fluentvalidation.codeplex.com), and I have written [a number of posts ](/tags/fluent%20validation/)about it before. On the current project I am working we needed to do database validation which I described how to do in [this blog post](http://www.jerriepelser.com/blog/using-fluent-validation-with-asp-net-mvc-part-4-database-validation/).  This time however I needed to go one step further and not only do the database validation on the server side, but also on the client side.

FluentValidation integrates with the ASP.NET model validation pipeline, and even go so far as to render client side validation for certain validation rules. More information on the ASP.NET Integration can be found on [this page](http://fluentvalidation.codeplex.com/wikipage?title=mvc) on the Wiki.  

The client side validation in ASP.NET MVC (currently at version 5) is done using jQuery validation and you can also allow for unobtrusive validation which output the validation parameters as HTML `data-` attributes and then gets parsed up by the jQuery Validation engine when the page renders. If you want a better understanding of how unobtrusive validation in ASP.NET MVC works, I suggest you go read Brad Wilson's excellent blog post entitled [Unobtrusive Client Validation in ASP.NET MVC 3](http://bradwilson.typepad.com/blog/2010/10/mvc3-unobtrusive-validation.html).

Adding an unobtrusive validator in ASP.NET MVC using FluentValidation is a fairly straightforward process which I will explain in the rest of this blog post. 

I am using the Autofac for dependency injection and I am not going to cover that in detail again. If you want to understand that then review my previous blog posts on [using dependency injection with FluentValidation](/blog/using-fluent-validation-with-asp-net-mvc-part-3-adding-dependency-injection/), and [adding database validation to FluentValidation](http://www.jerriepelser.com/blog/using-fluent-validation-with-asp-net-mvc-part-4-database-validation/). As a matter of fact, it is best you review [part 1](http://www.jerriepelser.com/blog/using-fluent-validation-with-asp-net-mvc-part-1-the-basics/) and [part 2](http://www.jerriepelser.com/blog/using-fluent-validation-with-asp-net-mvc-part-2-unit-testing/) of that blog post series as well, as this blog post builds on top of all of them and I am not going to cover a lot of the details which I have covered in those four blog posts.

## Adding the email validator
In my previous post on FluentValidator I did the database validation using the `AbstractValidator.Custom` method. In order to do the remote validation I will need to write a proper custom validator by inheriting from the `PropertyValidator` class. More information on the options available to you when writing customer validators in FluentValidation is covered in the Wiki document entitled [Custom Validators](http://fluentvalidation.codeplex.com/wikipage?title=Custom). 

First let us review the view model for registering a new user:

``` csharp
public class RegisterViewModel
{
    [DataType(DataType.EmailAddress)]
    [Display(Name = "Email")]
    public string Email { get; set; }

    [DataType(DataType.Password)]
    [Display(Name = "Password")]
    public string Password { get; set; }

    [DataType(DataType.Password)]
    [Display(Name = "Confirm password")]
    public string ConfirmPassword { get; set; }
}
```

For the validator I create a new class which inherits from `PropertyValidator` and overrides the `IsValid` property to query my database context to see whether a user with the email address is already registered.

```csharp
public class UniqueEmailValidator : PropertyValidator
{
    private readonly Func<IApplicationDbContext> dbContextFunction;

    public UniqueEmailValidator(Func<IApplicationDbContext> dbContextFunction)
        : base("Email address is already registered")
    {
        this.dbContextFunction = dbContextFunction;
    }

    protected override bool IsValid(PropertyValidatorContext context)
    {
        IApplicationDbContext dbContext = dbContextFunction();
        string emailAddress = context.PropertyValue as string;

        ApplicationUser user = dbContext.Users.FirstOrDefault(u => u.Email == emailAddress);
        return user == null;
    }
}
``` 

And finally I create my validation class for the view model. Notice the use of the `UniqueEmailValidator` on the email property.

```csharp
public class RegisterViewModelValidator : AbstractValidator<RegisterViewModel>
{
    private readonly Func<IApplicationDbContext> dbContextFunction;
    public RegisterViewModelValidator(Func<IApplicationDbContext> dbContextFunction)
    {
        this.dbContextFunction = dbContextFunction;

        RuleFor(m => m.Email)
            .NotEmpty()
            .EmailAddress()
            .SetValidator(new UniqueEmailValidator(dbContextFunction));
        RuleFor(m => m.Password)
            .NotEmpty()
            .Length(6, 100);
        RuleFor(m => m.ConfirmPassword)
            .Equal(m => m.Password);
    }
}
```

> Just a quick side note at this point. If you are curious as to why I am injecting `Func<IApplicationDbContext>` into my validators instead of `IApplicationDbContext`, the explanation can be found in [this blog post](http://robdmoore.id.au/blog/2013/03/23/resolving-request-scoped-objects-into-a-singleton-with-autofac/). 

## Adding the remote validator
At this point when a user tries to register with an email address which has been registered before, the email address validator will pick it up and fail the validation. What is left is to hook into jQuery unobtrusive validation framework.

For that we need to create a class which inherits from `FluentValidationPropertyValidator` and specify the correct client validation rule. You can write your own client validation rules in javascript, but for our purposes we are going to make use of the existing `remote` validator.

```csharp
public class UniqueEmailPropertyValidator : FluentValidationPropertyValidator
{
    public UniqueEmailPropertyValidator(ModelMetadata metadata, ControllerContext controllerContext, PropertyRule rule, IPropertyValidator validator)
        : base(metadata, controllerContext, rule, validator)
    {
    }

    public override IEnumerable<ModelClientValidationRule> GetClientValidationRules()
    {
        if (!this.ShouldGenerateClientSideRules())
            yield break;

        var formatter = new MessageFormatter().AppendPropertyName(Rule.PropertyName);
        string message = formatter.BuildMessage(Validator.ErrorMessageSource.GetString());

        var rule = new ModelClientValidationRule
        {
            ValidationType = "remote",
            ErrorMessage = message
        };
        rule.ValidationParameters.Add("url", "/api/validation/uniqueemail");

        yield return rule;
    }

}
``` 

This will generate the correct `data-val-` attributes for the unobtrusive validation. At runtime the validation will be hooked up using the remote validation adapter.

Next we need to hook this property validator into the FluentValidation framework. We do that in the `AutofacConfig` class where we configure FluentValidation. Notice the call to `provider.Add(typeof(UniqueEmailValidator)....`. This associates the client validation with our `UniqueEmailValidator`, so the correct client validation codes get rendered. 

```csharp
FluentValidationModelValidatorProvider.Configure(provider =>
{
    provider.ValidatorFactory = new AutofacValidatorFactory(container);
    provider.AddImplicitRequiredValidator = false;

    provider.Add(typeof(UniqueEmailValidator), (metadata, context, description, validator) => new UniqueEmailPropertyValidator(metadata, context, description, validator));
});

```

The last bit is to write the ASP.NET API Controller which will be called by the remote validator to validate for the existence of an email address. For use with the remote validator this API method should simply return `true` or `false` to indicate success or failure.

```csharp
[RoutePrefix("api/validation")]
public class ValidationController : ApiController
{
    private readonly IApplicationDbContext dbContext;

    public ValidationController(IApplicationDbContext dbContext)
    {
        this.dbContext = dbContext;
    }

    [Route("uniqueemail")]
    [HttpGet]
    public async Task<IHttpActionResult> ValidateUniqueEmail(string email)
    {
        bool validationResult = true;

        if (!String.IsNullOrEmpty(email))
        {
            var user = await dbContext.Users.Where(u => u.Email == email)
                                        .FirstOrDefaultAsync();

            validationResult = (user == null);
        }

        return Ok(validationResult);
    }

}
```

## Running the application
All the pieces are now in place and we can test the application. Start the application and navigate to the the register form.

When you inspect the email input field you will notice the unobtrusive validation attributes being rendered.  

![](/assets/images/remote-client-side-validation-with-fluentvalidation/runtime.png)

When you type and email address into the email address field the remote validation will automatically make a call to the remote API method to validate the email address as soon as the email input fields loses focus.

![](/assets/images/remote-client-side-validation-with-fluentvalidation/validation-failed.png)
