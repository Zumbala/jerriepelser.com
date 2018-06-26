---
date: 2013-05-13T00:00:00Z
description: |
  Part 2 of 4 of my introduction to using FluentValidation in ASP.NET MVC. This post covers how to unit test your validation classes.
tags:
- aspnet mvc
- fluent validation
- validation
title: 'Using Fluent Validation with ASP.NET MVC - Part 2: Unit Testing'
url: /blog/using-fluent-validation-with-asp-net-mvc-part-2-unit-testing/
---

In my [previous blog](/blog/using-fluent-validation-with-asp-net-mvc-part-1-the-basics/) post I looked at the basics of setting up and using [FluentValidation](http://fluentvalidation.codeplex.com) with ASP.NET MVC.  This time around we will be having a look at unit testing with FluentValidation.  There are a few approaches to do validation of your model or view model objects using data annotations.  One of the ways to do this is by using the approach described by K. Scott Allen in [this blog post](http://odetocode.com/Blogs/scott/archive/2011/06/29/manual-validation-with-data-annotations.aspx).  FluentValidation makes the process of unit testing your validation errors quite easy.

## Fluent Validation Helpers

The FluentValidation framework supplies us with a few helper methods to assist in testing validation rules which are located in the FluentValidation.TestHelper namespace.

``` csharp
public static void ShouldHaveValidationErrorFor<T, TValue>(this IValidator<T> validator, Expression<Func<T, TValue>> expression, TValue value)
public static void ShouldHaveValidationErrorFor<T, TValue>(this IValidator<T> validator, Expression<Func<T, TValue>> expression, T objectToTest)
public static void ShouldNotHaveValidationErrorFor<T, TValue>(this IValidator<T> validator, Expression<Func<T, TValue>> expression, TValue value)
public static void ShouldNotHaveValidationErrorFor<T, TValue>(this IValidator<T> validator, Expression<Func<T, TValue>> expression, T objectToTest)
```

## Validating individual properties

These methods allow us to easily test each of the validation rules individually without the need to construct a complete test object to validate each time.  For example, our unit test to ensure that the username gets validated correctly for null values looks as follows:

``` csharp
[TestMethod]
public void ShouldHaveValidationErrorWhenNoUsernameIsSupplied()
{
    validator.ShouldHaveValidationErrorFor(x => x.UserName, (string)null);
}
```

As you can see we do not have to construct an entire test RegisterModel object, but just have to specify the property which we want to validate and the value we want to validate against.  The example above is a negative test and ensures that a validation error gets generated when a property value is not valid.  We can also test to ensure that no validation error gets generated when we pass a valid value:

``` csharp
[TestMethod]
public void ShouldNotHaveValidationErrorWhenUsernameIsSupplied()
{
    validator.ShouldNotHaveValidationErrorFor(x => x.UserName, "username");
}
```

## Validating related properties

Lastly we can also pass in an entire object to validate, for example when we want to validate that the Password and ConfirmPassword properties match

``` csharp
[TestMethod]
public void ShouldHaveValidationErrorWhenPasswordAndConfirmPasswordDoesNotMatch()
{
    validator.ShouldHaveValidationErrorFor(x => x.ConfirmPassword,
        new RegisterModel
        {
            Password = "password1",
            ConfirmPassword = "password2"
        });
}
```

## Validating an entire object

All the above helper methods test only for the case where we want to test for a validation error related to a certain property.  We  can also test to ensure that the entire object is valid or when we want to test for validation errors on the object level, i.e. validation errors which relate to the object as a whole and not to specific properties.  In such cases we can simply call the Validate method on our Validator which will return a ValidationResult.

The ValidationResult contains a property IsValid which indicates whether the object is valid, as well as an Errors property which contains a collection of errors when the object is not valid.  As an example we pass in a valid object, run in through the validation and ensure that the validation result is valid:

``` csharp
[TestMethod]
public void ShouldNotHaveValidationErrorsWhenValidModelIsSupplied()
{
    ValidationResult validationResult = validator.Validate(new RegisterModel
                                                            {
                                                                UserName = "username",
                                                                Password = "password",
                                                                ConfirmPassword = "password"
                                                            });
    validationResult.IsValid.Should()
                    .BeTrue();
}
```

In case you wondering about the manner in which I do the assertion, I am using the wonderful [FluentAssertions ](http://fluentassertions.codeplex.com/)class library in the example above.

[Next time](/blog/using-fluent-validation-with-asp-net-mvc-part-3-adding-dependency-injection/) we will look at one of the main reasons I use the FluentValidation library, which is the ease with which we can do dependency injection.