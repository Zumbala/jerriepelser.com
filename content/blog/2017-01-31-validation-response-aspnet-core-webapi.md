---
date: 2017-01-31T00:00:00Z
description: |
  Shows how you can alter the response sent from your Web API in response to validation errors
tags:
- asp.net core
- asp.net
- fluentvalidation
title: Handling validation responses for ASP.NET Core Web API
url: /blog/validation-response-aspnet-core-webapi/
---

I have been working on project where one of the things I needed to handle was returning a response when model validation fails when calling any of my API endpoints.

## The quick way

Thankfully the [ASP.NET Filter Documentation](https://docs.microsoft.com/en-us/aspnet/core/mvc/controllers/filters) contains a nice sample of how to do this. Basically all you need to do is to create a custom Action Filter which will check if the `ModelState` is valid, and if not it returns a `BadRequestObjectResult` containing the `ModelState`.

So create the attribute:

```csharp
public class ValidateModelAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext context)
    {
        if (!context.ModelState.IsValid)
        {
            context.Result = new BadRequestObjectResult(context.ModelState);
        }
    }
}
```

Hook it up to your Controller:

```csharp
[Authorize]
[Route("api/properties")]
[ValidateModel]
public class PropertiesController : Controller
{
    // code omitted for brevity
}
```

Now every time I make a call to one of the actions in that controller, and the model validation fails, it will automatically return 400 (Bad Request) response, and the body of the response will contain the errors, for example:

```json
{
  "Url": [
    "'Url' should not be empty."
  ]
}
```

> Also read Steve Smith's MSDN article on [Real World ASP.NET MVC Filters](https://msdn.microsoft.com/en-us/magazine/mt767699.aspx)

## The custom way

This is however not exactly how I want my error responses formatted. I want a standard error response structure I would like to use throughout my API. For example, if an exception occurs in my application I would like to return something like the following:

```json
{ "message" : "Problems parsing JSON"}
```

More importantly for the purpose of this blog post, if there is actual validation errors, I would rather want to return a 422 (Unprocessable Entity) and I want the body to be something like the following:

```json
{
  "message": "Validation Failed",
  "errors": [
    {
      "field": "Url",
      "message": "'Url' should not be empty."
    }
  ]
}
```

So the structure will contain a `message` attribute which for validation errors will simply be "Validation Failed". Then there is an `errors` attribute which contains an array of errors. Each error element contains the `field` to which the error relates to, as well as the `message` for the error.

In the case of model-wide validation errors there will be no `field` specified, so it can look something like the following:

```json
{
  "message": "Validation Failed",
  "errors": [
    {
      "message": "This is a model-wide error"
    },
    {
      "field": "Url",
      "message": "'Url' should not be empty."
    }
  ]
}
```

So to do the was easy. First I created a class for the response I want to return:

```csharp
public class ValidationError
{
    [JsonProperty(NullValueHandling=NullValueHandling.Ignore)]
    public string Field { get; }

    public string Message { get; }

    public ValidationError(string field, string message)
    {
        Field = field != string.Empty ? field : null;
        Message = message;
    }
}

public class ValidationResultModel
{
    public string Message { get; } 

    public List<ValidationError> Errors { get; }

    public ValidationResultModel(ModelStateDictionary modelState)
    {
        Message = "Validation Failed";
        Errors = modelState.Keys
                .SelectMany(key => modelState[key].Errors.Select(x => new ValidationError(key, x.ErrorMessage)))
                .ToList();
    }
}
```

Notice the `[JsonProperty(NullValueHandling=NullValueHandling.Ignore)]` attribute on the `Field` property. This is to ensure that the field will not be serialized in the case of a null value - i.e. for model-wide validation errors.

The `ModelStateDictionary` from which I obtain the errors does not allow null values for the `Key`, but it does allow empty strings. I want to convert this to a `null` in the case of an empty string, so the `Field` property does not get serialized for empty field names. That is the check I do in the constructor for `ValidationError`.

Also note that in the constructor of `ValidationResultModel` I take the errors and I flatten it. For the `ModelStateDictionary` the `Keys` property will contain a key value, which is typically the property name, and the the `Errors` property for that key will contain all the errors related to that field.

I want to flatten this structure to simple key-value pairs. So if a `Key` contains 2 `Errors`, it will be flattened to 2 `ValidationError` entries - one for each error.

The last thing that remains is to create my own custom `IActionResult` which I will return. I do not want to return `BadRequestObjectResult` because that returns an HTTP Status Code 400, and I want to return a 422 instead.

```csharp
public class ValidationFailedResult : ObjectResult
{
    public ValidationFailedResult(ModelStateDictionary modelState) 
        : base(new ValidationResultModel(modelState))
    {
        StatusCode = StatusCodes.Status422UnprocessableEntity;
    }
}
```

And the final step is to update my `ValidateModelAttribute` to instead return the new `ValidationFailedResult`:

```csharp
public class ValidateModelAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext context)
    {
        if (!context.ModelState.IsValid)
        {
            context.Result = new ValidationFailedResult(context.ModelState);
        }
    }
}
```

And that's all there is to it.
