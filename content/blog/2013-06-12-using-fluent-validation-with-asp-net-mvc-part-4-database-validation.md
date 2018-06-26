---
date: 2013-06-12T00:00:00Z
description: |
  Part 4 of 4 of my introduction to using FluentValidation in ASP.NET MVC. This post covers how to validation against your database.
tags:
- aspnet mvc
- fluent validation
- unit tests
- validation
title: 'Using Fluent Validation with ASP.NET MVC – Part 4: Database Validation'
url: /blog/using-fluent-validation-with-asp-net-mvc-part-4-database-validation/
---

In the [previous blog post](/blog/using-fluent-validation-with-asp-net-mvc-part-3-adding-dependency-injection/) we added dependency injection to our project and created our own validation factory which resolves the validators using dependency injection.  In this, the final blog post of the series, we will have a look at extending our existing RegisterModelValidator to do some database validation.  The database is accessed using the repository pattern which allows us to mock the repository and enable us to unit test the validator without a physical database.

The repository pattern I used in this sample is based on this [ASP.NET MVC Tutorial](http://www.asp.net/mvc/tutorials/getting-started-with-ef-using-mvc/implementing-the-repository-and-unit-of-work-patterns-in-an-asp-net-mvc-application).

## Creating the Repository

The first step is to create a simple repository called UserProfileRepository which is based on the interface IUserProfileRepository.  The reason for the interface is to allow us to mock and inject the repository easily.

``` csharp
public interface IUserProfileRepository
{
    void DeleteUserProfile(int userId);

    UserProfile GetUserProfileById(int userId);

    UserProfile GetUserProfileByUserName(string userName);

    IEnumerable<UserProfile> GetUserProfiles();

    void InsertUserProfile(UserProfile profile);

    void Save();

    void UpdateUserProfile(UserProfile profile);
}
```

You can find the code for the interface and the actual implementation in the Models folder of the web project.  The specific method of interest is GetUserProfileByUserName, which will retrieve a user by their user name and will allow us to validate whether a specific user name is already registered before we try and create the user.

## Extending the Validator

The next step is to extend our existing RegisterModelValidator to use the repository to query for an existing user name.

``` csharp
public class RegisterModelValidator : AbstractValidator<RegisterModel>
{
    private readonly IUserProfileRepository userProfileRepository;

    public RegisterModelValidator(IUserProfileRepository userProfileRepository)
    {
        this.userProfileRepository = userProfileRepository;

        RuleFor(x => x.UserName)
            .NotEmpty();
        RuleFor(x => x.Password)
            .NotEmpty()
            .Length(6, 100);
        RuleFor(x => x.ConfirmPassword)
            .Equal(x => x.Password);

        Custom(rm =>
                {
                    UserProfile userProfile = userProfileRepository.GetUserProfileByUserName(rm.UserName);

                    if (userProfile != null)
                        return new ValidationFailure("UserName", "This user name is already registered");

                    return null;
                });
    }
}
```

The first change is to pass an instance of IUserProfileRepository to the constructor and store it in an instance variable. The second step is to use the Custom() method of the AbstractValidator class of FluentValidation to query the repository for an existing user with that user name and if so, will return a ValidationFailure stating that the user name is already registered.

More information on writing custom validation rules using the Custom method of AbstractValidator can be found in [the FluentValidation documentation](http://fluentvalidation.codeplex.com/wikipage?title=Custom&referringTitle=Documentation&ANCHOR#AbstractValidatorCustom "Custom&referringTitle=Documentation&ANCHOR#AbstractValidatorCustom").

## Registering the Repository with the IOC container

We need to ensure that we register the UserProfileRepository class with Autofac so that it can be injected into the constructor of the RegisterModelValidator class at runtime.  To do this we will create a new Autofac module to register all our repository classes, exactly like we did in the previous blog post to register the validators.

``` csharp
public class RepositoryModule : Module
{
    protected override void Load(ContainerBuilder builder)
    {
        builder.Register(x => new UserProfileRepository(new UsersContext()))
                .As<IUserProfileRepository>();
    }
}
```

This will tell Autofac that every time it needs to inject an instance of the IUserProfileRepository interface it should create an instance of the actual UserProfileRepository class to pass along.  Of course we need to register this new module in our AutofacConfig class which was also created in the previous blog post.

``` csharp
public static void RegisterComponents()
{
    var builder = new ContainerBuilder();
    builder.RegisterControllers(typeof(MvcApplication).Assembly);
    builder.RegisterFilterProvider();

    // Register the modules
    builder.RegisterModule<RepositoryModule>();
    builder.RegisterModule<ValidationModule>();

    // Refer to the source code for the rest of the method
    // ....
}
```

At this point we can run the application and the validation will work, but before we do that let's extend the unit tests to ensure our new custom database validation works.

## Unit Testing the Validator

The final bit is to test the database validation code.  To do this we will need to mock the IUserProfileRepository interface, which will allow us to write our unit test without actually requiring a physical database.  For the mocking framework I decided to use [Moq](https://code.google.com/p/moq/), which I installed using Nuget.

![](/assets/images/2013/06/Capture.png)

Next we update the Setup() method of the RegisterModelValidatorTests class to create a mock instance of IUserProfileRepository and pass that to the RegisterModelValidator class.

``` csharp
public void SetUp()
{
    repository = new Mock<IUserProfileRepository>();
    repository.Setup(x => x.GetUserProfileByUserName("existing_username"))
                .Returns(new UserProfile());

    validator = new RegisterModelValidator(repository.Object);
}
```

Take note of lines 4 and 5 of the code above.  It states that whenever we call the GetUserProfileByUserName() method of the mock object with the value of "existing_username" for the userName parameter, the mock object should return an instance of the UserProfile class.  This basically emulates an existing user in the database with the user name "existing_username".  For more information on how to do mocking with Moq, please refer to their [Quickstart documentation](https://code.google.com/p/moq/wiki/QuickStart).

The last thing is to create a unit test which will ensure that is we validate with a user name of "existing_username" our validator whould return a validation error.

``` csharp
[TestMethod]
public void ShouldHaveValidationErrorWhenUserNameExists()
{
    validator.ShouldHaveValidationErrorFor(x => x.UserName, "existing_username");
}
```

## Conclusion

In this series we had a look at using FluentValidation with ASP.NET MVC.  In [Part 1](/blog/using-fluent-validation-with-asp-net-mvc-part-1-the-basics/) we looked at the basics of installing FluentValidation and replacing the existing data annotations validation with FluentValidation.  In [Part 2](/blog/using-fluent-validation-with-asp-net-mvc-part-2-unit-testing/) we looked at how easy it is to unit test our FluentValidation validators.  In [Part 3](/blog/using-fluent-validation-with-asp-net-mvc-part-3-adding-dependency-injection/) we added dependency injection with Autofac, and in this final blog post we brought everything together by adding database validation using a repository, injecting the repository at runtime and mocking the repository for our unit tests.
