---
date: 2013-05-26T00:00:00Z
description: |
  Part 3 of 4 of my introduction to using FluentValidation in ASP.NET MVC. This post covers how to inject dependencies into your validation classes.
tags:
- aspnet mvc
- autofac
- dependency injection
- fluent validation
- validation
title: 'Using Fluent Validation with ASP.NET MVC - Part 3: Adding Dependency Injection'
url: /blog/using-fluent-validation-with-asp-net-mvc-part-3-adding-dependency-injection/
---

In the [previous blog post](/blog/using-fluent-validation-with-asp-net-mvc-part-2-unit-testing/) we looked at how to do unit testing with FluentValidation. My goal is to show you how we can write validators which can validate information in the database, but still be easy to unit test.  For this we will make use of dependency injection, mocking and the repository pattern.  Before we can write and unit test the actual database validator we need to put dependency injection in place as it will make things a bit easier for us going ahead.  In this blog post I will illustrate how to add dependency injection to our existing project.

This article will assume that you have a familiarity with dependency injection and how it can be used in the ASP.NET MVC Framework.  If you are not familiar with it I would suggest that you work through the hands-on lab entitled [ASP.NET MVC 4 Dependency Injection](http://www.asp.net/mvc/tutorials/hands-on-labs/aspnet-mvc-4-dependency-injection).

## Installing Autofac

For the purposes of this blog post I will be using [Autofac](https://code.google.com/p/autofac/).  There are a lot if DI frameworks out there, each with each advantages and disadvantages, so I would suggest you use the one you are most comfortable with. If you are new to this, Daniel Palme maintains a [very nice benchmark and comparison](http://www.palmmedia.de/Blog/2011/8/30/ioc-container-benchmark-performance-comparison) of all the .NET dependency injection frameworks.  If you are just starting off with DI, I would however suggest that you use a framework for which there is decent documentation and other resources available on the internet.

Adding Autofac to out project is as simple as installing the Nuget package.  As I will be using this in an MVC 4 project I just install the package entitled "Autofac ASP.NET MVC 4 Integration".  This package depends on the core Autofac package, so it will install it automatically.

![](/assets/images/2013/05/Capture1.png)

## Integrating Autofac with MVC

Once Autofac is installed, the next step is to integrate it with the ASP.NET MVC 4 Framework.  The purpose is to ensure that our controllers and filters are resolved through Autofac.  This will allow us to easily inject dependencies into our controllers and filters, which will be resolved by Autofac at runtime.  I created a class called AutofacConfig in the App_Start folder which does all the setup and registration which is required.

``` csharp
public class AutofacConfig
{
    public static void RegisterComponents()
    {
        var builder = new ContainerBuilder();
        builder.RegisterControllers(typeof(MvcApplication).Assembly);
        builder.RegisterFilterProvider();

        // Create the container
        var container = builder.Build();

        // Set the dependency resolver for MVC
        DependencyResolver.SetResolver(new AutofacDependencyResolver(container));
    }
}
```

On line 5 I create the ContainerBuilder which will help me with doing the component registrations and building the actual DI container.  Line 6 registers all the controllers which are available in the assembly.  Line 7 register all the filters.  Line 10 creates the actual container.  Line 13 sets the dependency resolver which is used by the MVC framework to a new instance of the AutofacDependencyResolver class, which will resolve the dependencies based in the container created in line 10.

The final step is to ensure that the RegisterComponents method is called.  We do this by adding a call to AutofacConfig.RegisterComponents() in the Application_Start method of the Global.asax:

``` csharp
protected void Application_Start()
{
    AreaRegistration.RegisterAllAreas();

    WebApiConfig.Register(GlobalConfiguration.Configuration);
    FilterConfig.RegisterGlobalFilters(GlobalFilters.Filters);
    RouteConfig.RegisterRoutes(RouteTable.Routes);
    BundleConfig.RegisterBundles(BundleTable.Bundles);
    AuthConfig.RegisterAuth();
    AutofacConfig.RegisterComponents();

    FluentValidationModelValidatorProvider.Configure();
}
```

## Creating a custom Validator Factory

We have introduced Dependency Injection into the project, but our validators are still not resolved through the DI framework.  The way we do that is by writing our own ValidatorFactory for FluentValidation.  You can read more about this in [the documentation on their website](http://fluentvalidation.codeplex.com/wikipage?title=ValidatorFactory&referringTitle=Documentation "ValidatorFactory&referringTitle=Documentation").

The actual code for the validator factory is quite simple.  We pass in the Autofac container into the constructor and simple resolve the actual validator from the container when the FluentValidation library requests it.  One thing to note is that the FluentValidation library request a validator by its generic type.  For example when it is looking for the validator for our RegisterModel class, the validatorType parameter which gets passed in to the CreateInstance method will be for `IValidator<RegisterModel>`.  This is important to remember because when we will register the various validators in Autofac, we will use this type as the Key.  More on this below.

``` csharp
public class AutofacValidatorFactory : ValidatorFactoryBase
{
    private readonly IContainer container;

    public AutofacValidatorFactory(IContainer container)
    {
        this.container = container;
    }

    public override IValidator CreateInstance(Type validatorType)
    {
        IValidator validator = container.ResolveOptionalKeyed<IValidator>(validatorType);
        return validator;
    }
}
```

The next step is to register the validator factory with FluentValidation, as well as register FluentValidation as the model validator provider for the MVC framework.  To do this we add the following lines to our AutofacConfig class:

``` csharp
// Set up the FluentValidation provider factory and add it as a Model validator
var fluentValidationModelValidatorProvider = new FluentValidationModelValidatorProvider(new AutofacValidatorFactory(container));
DataAnnotationsModelValidatorProvider.AddImplicitRequiredAttributeForValueTypes = false;
fluentValidationModelValidatorProvider.AddImplicitRequiredValidator = false;
ModelValidatorProviders.Providers.Add(fluentValidationModelValidatorProvider);
```

Line 2 creates a new FluentModelValidatorProvider (which inherits from the MVC framework ModelValidatorProvider class) and passes in our new AutofacValidatorFactory as the factory class it will use.  For more detail on how the ModelValidatorProvider class work, you can refer to [this blog post](http://bradwilson.typepad.com/blog/2010/10/service-location-pt6-model-validation.html) by Brad Wilson.

Line 3 and 4 simply disables the implicit required validator being added for both the DataAnnotationsModelProvider, as well as the FluentValidationModelValidatorProvider.  If you do not add this then the respective model validator providers will automatically add "required" validators for any non-nullable type.  You can leave it enabled if you want to, it is up to you.  I like to be more explicit about my validators.

Line 5 simply adds the new model validator provider to the list of model validator providers used by the MVC framework.

**Please note** that the the MVC Framework will by default register a few model validator providers, such as the DataAnnotationsModelValidatorProvider.  If you do not want to use these you can simple clear the default list by making a call to ModelValidatorProviders.Providers.Clear() before adding the FluentValidationModelValidatorProvider to the list.

## Registering our Validators

Now that we have registered our own ValidatorFactory, we need to register the validators.  To do this I create my own Autofac module to handle the registration of the validators.  More info on modules can be found [on the Autofac website.](https://code.google.com/p/autofac/wiki/StructuringWithModules)

``` csharp
public class ValidationModule : Module
{
    protected override void Load(ContainerBuilder builder)
    {
        builder.RegisterType<RegisterModelValidator>()
                .Keyed<IValidator>(typeof(IValidator<RegisterModel>))
                .As<IValidator>();
        builder.RegisterType<RegisterExternalLoginModelValidator>()
                .Keyed<IValidator>(typeof(IValidator<RegisterExternalLoginModel>))
                .As<IValidator>();
        builder.RegisterType<LoginModelValidator>()
                .Keyed<IValidator>(typeof(IValidator<LoginModel>))
                .As<IValidator>();
        builder.RegisterType<LocalPasswordModelValidator>()
                .Keyed<IValidator>(typeof(IValidator<LocalPasswordModel>))
                .As<IValidator>();
    }
}
```

One thing to take note of is how I register each of my validators.  Remember that I mentioned that FluentValidation will request the validator by the type of the validator, for example when it requests the validator for the RegisterModel class it will request it by the type IValidator<RegisterModel>.  In my AutofacValidatorFactory class I request it from the container using this type as the key.  I therefore need to register the appropriate validator using this type as the key.  You will notice in the class above that I for example register the RegisterModelValidator class by the key IValidator<RegisterModel>.

The final bit of the puzzle is to register this module in our AutofacConfig class:

``` csharp
// Register the modules
builder.RegisterModule<ValidationModule>();
```

## Some cleanup

At this point we can run the project and everything should work, but there is a bit of cleanup we can do in the code.

The first bit of code we can remove is the call to FluentValidationModelValidatorProvider.Configure() in the Application_Start method of Global.asax.  This is not needed anymore as all it really does is set up its own FluentValidationModelValidatorProvider which resolves the validator for a class through the FluentValidation Validator attribute class which we specified previously on our models.

Seeing as it is not using these attributes anymore, you can now also go back to the models and remove the Validator attribute from each of the model classes.

## Conclusion

In this blog post we had a look at how to introduce dependency injection into the project and ensure that our validators are resolved through the DI container.  [Next time](/blog/using-fluent-validation-with-asp-net-mvc-part-4-database-validation/) we will look at how to create a validator which does some validation by querying the database.  We will use a simple repository pattern and dependency injection to inject the repository into the validator.