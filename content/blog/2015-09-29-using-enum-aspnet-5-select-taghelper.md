---
date: 2015-09-29T00:00:00Z
description: |
  The new select TagHelper allows you to bind to an enumerated type, and I show you how you can display a list of possible values in the Select list.
tags:
- asp.net
- asp.net 5
- asp.net mvc
- taghelpers
title: Using enums with the ASP.NET 5 (MVC 6) Select TagHelper
url: /blog/using-enum-aspnet-5-select-taghelper/
---

## Introduction

In a previous blog post I demonstrated the [support for option groups in the new Select TagHelper in MVC 6](http://www.jerriepelser.com/blog/using-option-groups-with-mvc6-select-taghelper). Someone asked about using enums with the Select TagHelper, so let us look at how you can use enums with the TagHelper.

**This blog post is based on ASP.NET 5 Beta 7.** 

## Basic binding 

For demonstration purposes, let us assume that we have an enum type with the list of possible relationship types:

``` csharp
public enum RelationshipStatus
{
    Single,
    Complicated,
    Open,
    Relationship,
    Engaged,
    Married,
    CivilUnion,
    Separated,
    Divorced,
    Widowed
}
```

And we also have a ViewModel defined which our form will bind to:

```csharp
public class PersonalDetailsViewModel
{
    // Other properties omitted for brevity

    public RelationshipStatus RelationshipStatus { get; set; }
}
```

To bind this property to a select we will need to first specify the `model` of our Razor view:

``` csharp 
@model PersonalDetailsViewModel
``` 

And then we can specify the select and bind the select to the property by using the `asp-for` attribute:

``` html
<select asp-for="RelationshipStatus">
</select>
```

A reader has pointed out to me that apparently ASP.NET does not do data binding properly when using the shorthand ending tag for the SELECT element. In other words **do not use** `<select/>`, but instead make sure to use `<select></select>`.
{: .notice--warning}

The problem is however that the select list will be empty as we have not specified any options that the user can select from. To remedy this we can specify the options for the `<select>` inside the HTML:
 
``` html
<select asp-for="RelationshipStatus">
    <option value="0">Single</option>
    <option value="1">It is Complicated</option>
    <!-- etc..... -->
</select>
```

This will certainly work, but it is not ideal because we will need to updated all our views every time we add a new value to the enum.

Thankfully there is a `GetEnumSelectList` in the MVC 6 framework which is specified on the `IHtmlHelper` interface. You can see [the actual implementation](https://github.com/aspnet/Mvc/blob/7b433820b17c9a71f6924e95c746a1339fe439d3/src/Microsoft.AspNet.Mvc.ViewFeatures/ViewFeatures/HtmlHelper.cs#L346-L360) on the `HtmlHelper` class:

``` csharp
public IEnumerable<SelectListItem> GetEnumSelectList<TEnum>() where TEnum : struct
{
    var type = typeof(TEnum);
    var metadata = MetadataProvider.GetMetadataForType(type);
    if (!metadata.IsEnum || metadata.IsFlagsEnum)
    {
        var message = Resources.FormatHtmlHelper_TypeNotSupported_ForGetEnumSelectList(
            type.FullName,
            nameof(Enum).ToLowerInvariant(),
            nameof(FlagsAttribute));
        throw new ArgumentException(message, nameof(TEnum));
    }

    return GetEnumSelectList(metadata);
}
```

An instance of `IHtmlHelper` is already available in your Razor views through the `Html` property, so we can simply refer to it inside the view and use it in the `asp-items` attribute to generate the list of items:

``` html
<select asp-for="RelationshipStatus" asp-items="Html.GetEnumSelectList<RelationshipStatus>()">
</select>
``` 

Now when we run the application and view the source code for the page, we will see that the following HTML was generated:

``` html
<select data-val="true" data-val-required="The RelationshipStatus field is required." id="RelationshipStatus" name="RelationshipStatus">
	<option selected="selected" value="0">Single</option>
	<option value="1">Complicated</option>
	<option value="2">Open</option>
	<option value="3">Relationship</option>
	<option value="4">Engaged</option>
	<option value="5">Married</option>
	<option value="6">CivilUnion</option>
	<option value="7">Separated</option>
	<option value="8">Divorced</option>
	<option value="9">Widowed</option>
</select>
```  

## Better text descriptions

The one problem above is that the text descriptions generated are not very "user friendly". We cannot display something like "Complicated" to the user. Instead we want to display a more friendly description such as "It is complicated".

To do that all you need to do is to specify the `[Display]` attribute which is in the `System.ComponentModel.DataAnnotations` namespace on the relevant enum values. After doing that this is what our enum definition looks like:

``` csharp
public enum RelationshipStatus
{
    Single,
    [Display(Name = "It is complicated")]
    Complicated,
    [Display(Name = "In an open relationship")]
    Open,
    [Display(Name = "In a relationship")]
    Relationship,
    Engaged,
    Married,
    [Display(Name = "In a Civil Union")]
    CivilUnion,
    Separated,
    Divorced,
    Widowed
}
``` 

Refresing the web page and looking at the generated source code again, we can see that we now have better description which are being displayed to to user:

``` html
<select data-val="true" data-val-required="The RelationshipStatus field is required." id="RelationshipStatus" name="RelationshipStatus">
	<option selected="selected" value="0">Single</option>
	<option value="1">It is complicated</option>
	<option value="2">In an open relationship</option>
	<option value="3">In a relationship</option>
	<option value="4">Engaged</option>
	<option value="5">Married</option>
	<option value="6">In a Civil Union</option>
	<option value="7">Separated</option>
	<option value="8">Divorced</option>
	<option value="9">Widowed</option>
</select>
``` 

## Using a Nullable type

One more change you may want to make is to allow the user not to specify a value. For this we update the underlying model and change the `RelationshipStatus` property to a nullable type:

``` csharp
public class PersonalDetailsViewModel
{
    // Other properties omitted for brevity

    public RelationshipStatus? RelationshipStatus { get; set; }
}
```

We also want to add a "not specified" option to the select list. Thankfully the Select TagHelper is quite clever in that it will take into account any `<option>` elements which are already specified on the `<select>` element and leave them alone. So we can change the existing markup to add a "not specified" option with an empty value:

``` html
<select asp-for="RelationshipStatus" asp-items="Html.GetEnumSelectList<RelationshipStatus>()">
    <option value="">-- Not specified --</option>
</select>
```

And once again, if we look at the generated HTML we can still see all the possible values for the enum, but now we also have the "-- Not Specified --" option which the user can select from when they do not want to specify a value:

``` html
<select id="RelationshipStatus" name="RelationshipStatus">
	<option value="">-- Not specified --</option>
	<option value="0">Single</option>
	<option value="1">It is complicated</option>
	<option value="2">In an open relationship</option>
	<option value="3">In a relationship</option>
	<option value="4">Engaged</option>
	<option value="5">Married</option>
	<option value="6">In a Civil Union</option>
	<option value="7">Separated</option>
	<option value="8">Divorced</option>
	<option value="9">Widowed</option>
</select>
```

## Conclusion

In this blog post I demonstrated how you can bind a `<select>` element to enum values using the new Select TagHelper in MVC. I also demonstrated a couple of ways in which you can add the possible values that a user can select from dropdown list.
 