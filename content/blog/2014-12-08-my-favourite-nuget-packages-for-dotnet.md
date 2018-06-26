---
date: 2014-12-08T00:00:00Z
description: |
  A list of all my favourite Nuget packages which I seem to use over and over again
tags:
- nuget
- .net
title: My favourite Nuget packages for .NET development
url: /blog/my-favourite-nuget-packages-for-dotnet/
---

There are some Nuget packages which I just seem to come back to every single time, because they solve a particular problem for me and they do so in a way which resonates with me. I thought I would put together a list of all my favourite Nuget packages. Nuget can be used to add all sort of libraries to your project, such as Javascript or CSS frameworks, but this list will be purely for .NET development. I am also excluding the obvious (Microsoft) ones such as MVC, Entity Framework, etc. This is about all those other little gems.


## The stalwarts
These are the main Nuget packages I seem to use in every single project I do.

### Fluent Validation
Around 2 years ago (in the beginning of this blog) I started off a series of blog posts about integrating Fluent Validation into your ASP.NET MVC application. To this day those are still some of my most read blog posts and Fluent Validation remains my go-to library for adding validation to my ASP.NET MVC and Web API applications. The ease of use and the fact that I can inject dependencies - and therefore more easily unit test my validation classes - are the killer features for me.

```text
PM> Install-Package FluentValidation
```

**Find it on GitHub** at https://github.com/JeremySkinner/FluentValidation   

### NSubstitute

I used to be a big fan of Moq but at a stage a couple of years ago it seemed to become a bit stagnant and there was not much development happening on it, so I looked around for a replacement and found NSubstitute. I have used it ever since. Moq seems to have picked up momentum again in the meantime but I am sticking with NSubstitute as it has served me well.  Whenever I create a new test project this is the first package I install. 

```text
PM> Install-Package NSubstitute
```

**Find it on GitHub** at http://nsubstitute.github.io/

### Fluent Assertions 

The next package I install in a test project after NSubstitute is Fluent Assertions. The breadth of assertions you can do is breathtaking and saves me a lot of time.

```text
PM> Install-Package FluentAssertions
``` 

**Find it on GitHub** at https://github.com/dennisdoomen/fluentassertions

### Autofac

I used another IoC container before I used Autofac but I honestly cannot event remember which one it was, that is how long I have used Autofac and how much I like it. I am sure there are other other dependency injection libraries which are perhaps more nimble and faster but Autofac is plenty fast enough and it has been able to do everything I have ever asked of it. 

It is available as a PCL so I can use it pretty much anywhere, it has plenty of extension points and the number of integrations with other libraries means I don't have to spend time rolling my own integrations and can instead spend that time to add features to my application. Autofac also seems to be one of the handful of libraries which ASP.NET 5 [is supporting out of the box](https://github.com/aspnet/DependencyInjection/tree/dev/src).

```text
PM> Install-Package Autofac
```

**Find it on GitHub** at https://github.com/autofac/Autofac

## The tiny gems
I use these Nuget packages less often because they mostly solve very specific issues which are not present on every single project. They are however packages I highly recommend you look at

### Refit

I don't think I use any "official" API wrappers anymore. Whenever I need to integrate with a REST API I just install Refit and create a quick wrapper around the specific API calls I am interested in. I use [Postman](http://www.getpostman.com/) to test the API call, copy the JSON input or output from the API call to the clipboard and use Visual Studio's ["Paste JSON as classes"](http://blogs.msdn.com/b/webdev/archive/2012/12/18/paste-json-as-classes-in-asp-net-and-web-tools-2012-2-rc.aspx) feature to create strongly typed classes for the JSON and then create an interface with the API definition for Refit to use. And there you have a custom API wrapper in minutes!

The number one reason I use Refit is that because the API definition is defined as a C# interface it is super easy to always use it with dependency injection and also to mock the API for unit tests. The other big reason is that it is a PCL so I can use it pretty much anywhere.

```text
PM> Install-Package refit
``` 

**Find it on GitHub** at https://github.com/paulcbetts/refit

### Travis.UriTemplates

I became aware of this one just last week. Oh boy if I had a penny for every time I wished there was an easier way to construct URIs.... Well there is. If you have any need for building URIs then you better install this library fast. It is an awesome little gem! 

```text
PM> Install-Package Tavis.UriTemplates
```

**Find it on GitHub** at https://github.com/tavis-software/Tavis.UriTemplates

### NBuilder

Sometimes you need to build a bunch of sample data for unit testing or for seed data for a database. NBuilder makes the job of creating those a breeze.

If you want realistic looking data for seeding a database you can use it in conjunction with [Faker.NET](https://github.com/oriches/faker-cs) (though Faker.NET has not seen any enhancement for the past 2 year it still does its job well...)

```text
PM> Install-Package NBuilder
```

**Find it on GitHub** at https://github.com/garethdown44/nbuilder/

### NodaTime

If you have ever done time and timezone related programming you will know what a pain in the ass it can be. NodaTime tries to make that process a little bit better. You can read more about the [rationale behind NodaTime](http://nodatime.org/1.3.x/userguide/rationale.html). From a unit testing perspective NodaTime makes the process also a bit easier as it exposes an `IClock` interface to get the current time, which can be mocked or you can even use their own `FakeClock` class to assist with testing. Also, NodaTime was written by the great John Skeet himself.

```text
PM> Install-Package NodaTime
```

**Find it on Google Code** at https://code.google.com/p/noda-time/    

### Humanizer

Want to convert a time to something more meaningful like 'Updated 5 seconds ago'? Want to title case a string? Want to display more human-friendly names for enums? Want to pluralize strings? Humanizer have got you covered for those and a whole lot more. The list just goes on and on, so best you check out their project page on GitHub to see what it can do for you.

```text
PM> Install-Package Humanizer
```

**Find it on GitHub** at https://github.com/MehdiK/Humanizer 

### Parsing files

I am busy with a project where I need to process all sorts of files, specifically Markdown, HTML and YAML. Well there are packages to help you with all of those :)

For Markdown I seem to have settled on CommonMark.NET ([on GitHub](https://github.com/Knagis/CommonMark.NET/)):

```text
PM> Install-Package CommonMark.NET
```

For HTML parsing I use [AngleSharp](https://github.com/FlorianRappl/AngleSharp), but [HtmlAgilityPack](http://htmlagilitypack.codeplex.com/) is also awesome and a bit more mature and may be a better option for you.

```text
PM> Install-Package AngleSharp

PM> Install-Package HtmlAgilityPack
```

For parsing YAML, [YamlDotNet](https://github.com/aaubry/YamlDotNet) suits my needs perfectly as it is still being actively developed (many of the other ones aren't) and also allows me to stronly type YAML structures

```text
PM> Install-Package YamlDotNet
```

## Conclusion

That's it. This is not an exhaustive list of all the "best" Nuget packages, but merely the ones I find pretty useful on a regular basis. Special thanks to all the authors and contributors to those packages - you make my life so much easier! :)

Have a special Nuget package you use daily which you think is not getting the attention it should? Give me a shout on Twitter at [https://twitter.com/jerriepelser](https://twitter.com/jerriepelser) and let me know.