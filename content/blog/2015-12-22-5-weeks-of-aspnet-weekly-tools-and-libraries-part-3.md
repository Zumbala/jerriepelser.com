---
date: 2015-12-22T00:00:00Z
description: |
  Part 3 in a series of 5 blog posts which recaps the Tools and Libraries I covered during 2015 in ASP.NET Weekly
title: 5 Weeks of ASP.NET Weekly Tools and Libraries - Part 3
url: /blog/5-weeks-of-aspnet-weekly-tools-and-libraries-part-3/
---

I publish a weekly newsletter for ASP.NET Developers called [ASP.NET Weekly](http://www.aspnetweekly.com) which contains a curated list of the best ASP.NET related content I came across during the previous week. 

Each week I feature a tool or library which I think ASP.NET developers will find useful. This is part 3 in a 5 part series of blog posts during which I recap all the tools and libraries I featured during the past year.

I re-publish these with the exact descriptions I used in the newsletter, so see my commentary in that context...

## 21. [Stuntman](https://github.com/ritterim/stuntman)

Stuntman is a library for impersonating users during development leveraging .NET Claims Identity. Used primarily in web environments like ASP.&#8203;NET MVC, ASP.&#8203;NET Web Forms, and OWIN applications that serve HTML. This allows you to test different user scenarios that exist in your application with minimal friction. It also allows you to share those scenarios with other team members via source control.

For more background on why this package was created, you can read Khalid Abuhakmeh's [introductory blog post](http://www.khalidabuhakmeh.com/use-stuntman-for-development-authentication-in-owin)

## 22. [Glyphfriend](https://visualstudiogallery.msdn.microsoft.com/5fd24afb-b3b2-4cec-9b03-1cfcec6123aa)

Glyphfriend is an open-source Visual Studio 2015 extension to enhance the existing Intellisense to display preview glyphs for many of the common glyph-based font libraries like Font Awesome, Foundation, IonIcons, Bootstrap Glyphicons and more. I find this a really handy extension to be quickly able to get a preview of how the font icon I specified will look like. 

## 23. [Toastr](http://codeseven.github.io/toastr/)

Toastr is a JavaScript library which allows you to easily add "toast" notifications to your website so you can inform your users of status of actions. It supports success, warning, info and error messages and is very configurable in terms of where and how messages appear.

To get a better idea if its capabilities, check out the [Demo](http://codeseven.github.io/toastr/demo.html) 

## 24. [Serilog](http://serilog.net/)

There are many logging frameworks for .NET out there and they all do pretty much the same thing - provide diagnostics logging for applications to a variety of outputs. What makes Serilog stand out from the crowd is that it handles logging of structured data in a more intelligent fashion. So instead of just doing a ToString() on objects, it can log the properties of objects in a human-readable format.

## 25. [AdminLTE](https://github.com/almasaeed2010/AdminLTE)

AdminLTE is a free Admin Control Panel theme based on Bootstrap 3. Don't let the fact that this is a free theme fool you - I have used many premium commercial themes and the quality and breadth of this theme is comparable to most all of them. I actually think it even surpasses many commercial themes in term of quality. 

If you are in the market for a fancy Admin template for a new website you are developing, [have a look at the demo](https://almsaeedstudio.com/preview). It even comes with [very good documentation](https://almsaeedstudio.com/themes/AdminLTE/documentation/index.html) as well. 

## 26. [Metro UI CSS](http://metroui.org.ua/)

If you are a lover of Microsoft's "Metro" user interface and would like to have that reflect in the websites you develop, then Metro UI CSS is for you. It is a complete CSS framework which allows you to develop web applications with the Windows Metro style. It even contains CSS overrides for 3rd party JavaScript components such as Select2 and DataTables.

## 27. [Loaders.css](http://connoratherton.com/loaders)

Loaders is a a collection of loading animations written entirely in css. Each animation is limited to a small subset of CSS properties in order to avoid expensive painting and layout calculations.

## 28. [Select2](https://select2.github.io/)

Select2 is a dropdown list on steriods. It provides you with a select box with support for searching, tagging, remote data sets, infinite scrolling, and much more. 

Have a look at the [Examples Page](https://select2.github.io/examples.html) to get an idea of the capabilities, and also look at the Wiki for [how to use it with ASP.&#8203;NET MVC](https://github.com/select2/select2/wiki/.Net-MVC-Example).

## 29. [JSON API](http://jsonapi.org/)

JSON API is a specification for building JSON APIs. It defines a set of conventions around how requests should be made to the server, and how the server should respond to those requests.

If your are using ASP.&#8203;NET Web API, then this is a handy resource for you. There is also .NET implementation available called [JSONAPI.&#8203;NET](https://github.com/SphtKr/JSONAPI.NET) which allows you to implement a JSON API compliant API with ASP.&#8203;NET Web API.

## 30. [Awesomeplete](http://leaverou.github.io/awesomplete/)

Awesomplete is (yet another) autocomplete control which you can use in your applications and works in all modern browsers. It contains a wide range of features that are typical for these sort of controls, but what set Awesomplete apart from the rest is that it requires not external dependencies, which make it ideal when you want to keep the size of your website down to a minimum.