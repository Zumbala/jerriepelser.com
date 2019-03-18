---
title: "ModelState interfering with Model Binding in ASP.NET Core"
description:
  Describes an issue I ran into where ASP.NET Core apparently was not binding to my model correctly which turned out to be a case of ModelState interfering.
tags:
- asp.net core
---

In an application I am working on, I have a page where a user can invite other users to their account. This page has a simple form at the top allowing the user to invite a new user and then, below that, is a grid displaying the existing users.

![](/images/blog/2019-03-18-modelstate-overriding-model-binding-aspnet-core/form-with-grid.png)

I do not want to do full page post-back every time a user sends a new invitation, so I am making use of AJAX to handle posting the new user invitation form to my controller action.

This is the code for the controller action:

```csharp
[HttpPost]
public async Task<IActionResult> InviteUser(InviteUser.Command command)
{
    if (ModelState.IsValid)
    {
        var commandResult = await _mediator.Send(command);
        if (commandResult.Succeeded)
        {
            return PartialView("_InviteNewUserForm", new InviteUser.Command());
        }

        commandResult.ValidationResult.AddToModelState(ModelState, string.Empty);
    }

    var actionResult = PartialView("_InviteNewUserForm", command);
    actionResult.StatusCode = (int)HttpStatusCode.UnprocessableEntity;

    return actionResult;
}
```

The logic of the action itself is relatively straightforward:

1. I check to ensure that the ModelState is valid
1. If so, I execute the command (using [Mediatr](https://github.com/jbogard/MediatR))
1. Once command finishes, I check whether it succeeded and if so, I return a "blank" new invitation form as a partial view
1. If there was an error, I add the validation result from the command to the ModelState
1. Finally, at this point, there must be some validation error, so I re-render the partial view and return that to the browser with an HTTP Status 422 (Unprocessable Entry)

If there is a validation error, this works fine as you can see from the screenshot below:

![](/images/blog/2019-03-18-modelstate-overriding-model-binding-aspnet-core/validation-error.png)

The problem, however, came in when a new record was added successfully. Instead of a "blank" partial view being returned, it would return the partial view containing the same data which was just posted:

![](/images/blog/2019-03-18-modelstate-overriding-model-binding-aspnet-core/form-data-not-cleared.png)

This did not make sense to me, since I specifically pass an empty model when rendering the partial view:

```csharp
return PartialView("_InviteNewUserForm", new InviteUser.Command());
```

Af first, I thought I did something wrong either on the back-end when rendering the partial view, or on the front-end when replacing the existing form with the content of the new partial view. I double-checked everything and was happy that my code was functioning as intended.

In the end, it turned out to be an issue with the ModelState. When debugging the code, I noticed that the ModelState contains entries for the properties from the incoming model:

![](/images/blog/2019-03-18-modelstate-overriding-model-binding-aspnet-core/model-state.png)

I am not 100% sure about the data binding internals, but it appears that the data binding mechanisms will favour any values contained in the ModelState over the actual model being passed to the `PartialView()` method call. This is probably the correct behaviour in most cases, but in my case I did not want this behaviour.

Understanding the underlying issue, I worked around it by adding a call to `ModelState.Clear()` before the partial view is being rendered.

```csharp
[HttpPost]
public async Task<IActionResult> InviteUser(InviteUser.Command command)
{
    if (ModelState.IsValid)
    {
        var commandResult = await _mediator.Send(command);
        if (commandResult.Succeeded)
        {
            ModelState.Clear();
            return PartialView("_InviteNewUserForm", new InviteUser.Command());
        }

        commandResult.ValidationResult.AddToModelState(ModelState, string.Empty);
    }

    var actionResult = PartialView("_InviteNewUserForm", command);
    actionResult.StatusCode = (int)HttpStatusCode.UnprocessableEntity;

    return actionResult;
}
```

With this in place, the application functions correctly and the values from the "empty" model I am passing to the `PartialView` method call is being used to when rendering the partial view.