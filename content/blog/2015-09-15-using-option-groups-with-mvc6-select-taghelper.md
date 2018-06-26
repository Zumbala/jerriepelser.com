---
date: 2015-09-15T00:00:00Z
description: |
  Demonstrates the support of the new Select TagHelper in MVC for option groups (OptGroup)
tags:
- asp.net
- asp.net 5
- asp.net mvc
- taghelpers
title: Using Option Groups with the Select TagHelper in MVC 6
url: /blog/using-option-groups-with-mvc6-select-taghelper/
---

## Introduction

In MVC 6 we are switching away from the use of the classical HTML Helpers in our Razor views to use TagHelpers instead. Various posts have been written about the new TagHelpers in MVC 6. If you are not familiar with them, then a good place to start would be Dave Paquette's [Complete Guide to the MVC 6 Tag Helpers](http://blogs.msdn.com/b/cdndevs/archive/2015/08/06/a-complete-guide-to-the-mvc-6-tag-helpers.aspx) or Matthew Jones' [Tag Helpers in ASP.NET 5 - An Overview](http://www.exceptionnotfound.net/tag-helpers-in-asp-net-5-an-overview).

If you are adventurous and want to write your own TagHelpers, you can also check out Rick Anderson's great guide on [Authoring Tag Helpers](http://docs.asp.net/projects/mvc/en/latest/views/tag-helpers/authoring.html).

In this blog post I would like to discuss one particular feature of the Select TagHelper, and that is the built-in support for [Option Groups](http://www.w3schools.com/tags/tag_optgroup.asp).

## Overview of Option Groups

Option Groups (`<optgroup`), allows you to group related options in a drop-down list. 

To demonstrate this, let us take a normal select list which allows a user to select an item from a list of cars:

``` html
<select>
   <option value="audi">Audi</option>
   <option value="mercedes">Mercedes</option>
   <option value="saab">Saab</option>
   <option value="volvo">Volvo</option>
</select>
```

This will render something like the following:

![](/assets/images/using-option-groups-with-mvc6-select-taghelper/normal-select.png)

By using `<optgroup>`, we can group the list of cars by the country of manufacture:

``` html
<select>
   <optgroup label="German Cars">
      <option value="audi">Audi</option>
      <option value="mercedes">Mercedes</option>
   </optgroup>
   <optgroup label="Swedish Cars">
      <option value="saab">Saab</option>
      <option value="volvo">Volvo</option>
   </optgroup>
</select>
```

Now when the user selects an item from the dropdown, the items will be visibly grouped by the country of manufacture which may make the selection process easier for the user.

![](/assets/images/using-option-groups-with-mvc6-select-taghelper/grouped-select.png)

## Using the select TagHelper

To demonstrate the use of the TagHelper, let us first create a ViewModel to which the view will bind. Let us assume that we have a customer ViewModel and among a whole range of other fields it also contains a `Vehicle` property:

``` csharp
public class CustomerViewModel
{
    // Other properties removed for brevity...

    public string Vehicle { get; set; } 
}
```

To bind the select to the `Vehicle` property we first need to specify the `CustomerViewModel` as the type for the view's model, so at the top of the Razor view include:

``` html
@model CustomerViewModel
```

And then use `asp-for` attribute to bind the select to the value of the `Vehicle` property:

``` html
<select asp-for="Vehicle">
    <optgroup label="German Cars">
        <option value="audi">Audi</option>
        <option value="mercedes">Mercedes</option>
    </optgroup>
    <optgroup label="Swedish Cars">
        <option value="saab">Saab</option>
        <option value="volvo">Volvo</option>
    </optgroup>
</select>
```

Now when the control is rendered and we look at the source code for the element, we can confirm that the control is bound to the `Vehicle` property of the ViewModel buy inspecting the `name` and `id` attributes:

![](/assets/images/using-option-groups-with-mvc6-select-taghelper/asp-for-generated-attributes.png)

The next thing is to use the `asp-items` attribute of the Select TagHelper to also bind to the list of items a user can select from. First let us add a property to the ViewModel for the items and populate it with the list of items:

``` csharp
public class CustomerViewModel
{
    // Other properties removed for brevity...

    public string Vehicle { get; set; } 

    public List<SelectListItem> Vehicles { get; private set; }

    public CustomerViewModel()
    {
        var germanGroup = new SelectListGroup {Name = "German Cars"};
        var swedishGroup = new SelectListGroup { Name = "Swedish Cars" };

        Vehicles = new List<SelectListItem>
        {
            new SelectListItem
            {
                Value = "audi",
                Text = "Audi",
                Group = germanGroup
            },
            new SelectListItem
            {
                Value = "mercedes",
                Text = "Mercedes",
                Group = germanGroup
            },
            new SelectListItem
            {
                Value = "saab",
                Text = "Saab",
                Group = swedishGroup
            },
            new SelectListItem
            {
                Value = "volvo",
                Text = "Volvo",
                Group = swedishGroup
            }
        };
    }
}
```

In my example I manually add a hardcoded list of items, but it may be typical for you to generate the list from a database table or some other source. Notice that I have defined two `SelectListGroup` objects for the country of manufacture, and then assigned the relevant one to the `Group` property of a `SelectListItem`.

Next I need to bind my select element to the `Vehicles` property by using the `asp-items` attribute. I have also removed the `<optgroup>` and `<option>` elements from the HTML.

``` html
<select asp-for="Vehicle" asp-items="Model.Vehicles">
</select>
```

Notice that whereas I could simply specify the property name when using the `asp-for` attribute to bind the value of the element, you will need to specify the correct context when using the `asp-items` attribute. So when binding to a property on the model you will need to specify it be prefixing it with `Model.`, e.g. `Model.Vehicles` binds the list of items to the `Vehicles` property of the view's model. You can also bind it to a property on the ViewBag by using `ViewBag.Vehicles` as an example.

Now when I run the application I get the same list as before:

![](/assets/images/using-option-groups-with-mvc6-select-taghelper/taghelper-select.png)

Inspecting the HTML you will also notice that the TagHelper has generated the correct HTML markup for me with the `<optgroup>` and `<option>` elements:

![](/assets/images/using-option-groups-with-mvc6-select-taghelper/taghelper-markup.png)

## Conclusion

In this blog post I demonstrated how you can bind the option list of a `select` element to a property of your ViewModel. This is especially handy when the list of items are generated from a database table or other dynamic source. By allowing you to specify a `Group` for each item, you can also present the information in a more user friendly manner by visually grouping related items together. 