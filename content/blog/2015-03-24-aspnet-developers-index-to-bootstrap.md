---
date: 2015-03-24T00:00:00Z
description: |
  A few resources for ASP.NET developers to help you with learning and getting more value from Twitter Bootstrap.
tags:
- aspnet
- aspnet mvc
- css
- bootstrap
title: The ASP.NET Developer's guide to Bootstrap
url: /blog/aspnet-developers-index-to-bootstrap/
---

## Introduction

Twitter Bootstrap is one of the most widely used CSS frameworks at the moment and there is a massive supporting ecosystem in themes, components, tutorials and the likes. Since Visual Studio 2013, the standard ASP.NET project template has also been based on Twitter Bootstrap. Now, along with the popularity of Bootstrap there has also been the inevitable backlash from people complaining that also website now look the same. The framework itself has also grown quite large over time and inevitably contains a lot of excess CSS classes which you do not use in your applications.

The thing is however, that Bootstrap makes it easy for non-designer types like me to build websites which look semi-decent. Add to that the massive ecosystem built up around it, you can get many themes and components to use in your applications which allow you to have a consistent look through out.

This blog post serves as an overview to some Bootstrap resources for ASP.NET developers to help you get started with Bootstrap and get the most out of it. This is **not** meant as an exhaustive list of all the Bootstrap resources out there, but rather to try and give a few *quality* resources which will help you get the most from Bootstrap in your application.     

## Get started using Bootstrap 

As I mentioned, one of the quickest ways to get started is to just create a new project in Visual Studio 2013, as the default "ASP.NET Web Application" template includes Bootstrap by default. It does this by adding the [Bootstrap Nuget package](https://www.nuget.org/packages/bootstrap). The package contains the full Bootstrap CSS and JavaScript files (normal and minified versions), as well as the [Glyphicons](http://glyphicons.com/) Halflings icon font. During installation of the package, the files are places in the appropriate folders (`\Content`, `\Scripts` and `\fonts`) and the default bundles in the `BundleConfig` class is set up to include these.

### LESS Source
If the size of the default Bootstrap package is not to your liking, and you would like a little bit more control over what to include, you can opt to rather install the [Bootstrap Less Source](https://www.nuget.org/packages/Twitter.Bootstrap.Less/) Nuget package. This will install the LESS source for Bootstrap, and you can then create your own LESS file to include only the parts of Bootstrap you want, and use [Web Essentials' LESS compiler](http://vswebessentials.com/features/less) to create a CSS file for you. And also, because it you have the LESS source you can also more easily override the variables for some of the standards color options. 

### Bower
If you have embraced the new web development tools such as Bower, then you may choose to rather install the Bower package:

``` text
bower install bootstrap
```

You can then bring the required files in from the `bower_components` folder, by either referencing it directly or using [Grunt](http://gruntjs.com/) or [Gulp](http://gulpjs.com/) to process it in some other way.

### Manual

Of course you can also do it manually by going to the Bootstrap website, [downloading it](http://getbootstrap.com/getting-started/) and copying the files into your project folder structure. The Bootstrap website also has a feature which allows you to [customize the parts you want to download](http://getbootstrap.com/customize/).

### Bootstrap CDN

Actually, you do not even have to have a local copy of Bootstrap. You can simply reference it from the [Bootstrap CDN](http://www.bootstrapcdn.com/). It would perhaps be best to use the CDN in combination with the ASP.NET bundling and minification features. There is an article on the ASP.NET website [which explains how to do it](http://www.asp.net/mvc/overview/performance/bundling-and-minification) (look for the "Using a CDN" section). 

## Learning Bootstrap

### Bootstrap Website

OK, now that you have included Bootstrap in your project it is time to start understanding how to use it. The [Bootstrap website](http://getbootstrap.com/) itself contains a wealth of information and samples, so that would be a good starting point. For the most part going through the documentation and examples is enough, so make that your starting point.

### Learning the grid

The Bootstrap grid is especially powerful and allow web developers to easily develop responsive websites, so it is really important to understand how it works. There are two very good tutorials I can point to. The first is [Bootstrap 3 Grid Introduction](http://www.helloerik.com/bootstrap-3-grid-introduction) from Erik Flowers, and the second is [Exploring the Bootstrap 3.0 Grid System](http://fearlessflyer.com/exploring-the-bootstrap-3-0-grid-system/) from Fearless Flyer.

### Youtube videos

There are a wide range of video tutorials available on Youtube. If you look at the [playlists by CodersGuide](https://www.youtube.com/user/CodersGuide/playlists) you will notice a number of different series of tutorials by them. There is also a full [Code a Responsive Website with Twitter Bootstrap](https://www.youtube.com/playlist?list=PLUoqTnNH-2Xz_BUrjcahKWDhPcUj-FTOt) course available from Brad Hussey.

### Commercial courses

If terms of commercial training course, there are [a number of course available on Udemy](https://www.udemy.com/courses/search/?ref=home&q=bootstrap). Pluralsight also has a [Bootstrap 3 course](http://www.pluralsight.com/courses/bootstrap-3) available from author Shawn Wildermuth.

## Make it pretty

### Bootswatch

So once you have installed Bootstrap and learned how to use it, you will most probably want to make your side a little bit prettier and try and get rid of the stock Bootstrap look-and-feel. The easiest way is probably to [head over to Bootswatch](https://bootswatch.com/) and download one of the many swatches (or themes) which are available. These are not **complete** themes or templates, but mostly consist of different color schemes and perhaps different fonts. If you are not sure how to integrate this with your ASP.NET MVC project, [here is a StackOverflow question with detailed instructions](http://stackoverflow.com/questions/21839351/is-there-a-walkthrough-for-implementing-a-theme-from-bootswatch-or-wrapbootstrap).

### Paintstrap

[Paintstrap](http://paintstrap.com/) allows you to generate a custom color scheme for Bootstrap from either an [Adobe Kuler](https://color.adobe.com) Color Theme or a [COLOURlovers](http://www.colourlovers.com/) palette.  

### Quality free themes 

There are some really good quality free themes available on the internet which can give your application a professional look without costing you a cent. You can look at either [Drunken Parrot UI Kit](http://hoarrd.github.io/drunken-parrot-flat-ui/) (don't let the name fool you), or [Designmodo's Flat UI](http://designmodo.github.io/Flat-UI/).

### Paid options

In terms of paid options, I have always been a fan of the quality of templates available on [Themeforest](http://themeforest.net/). They have a wide range of themes for many platforms (WordPress, Tumblr, Ghost, etc), but they also have a massive selection of plain HTML themes. Most of the themes nowadays are based on Bootstrap, and when you look at the details for a particular theme they will indicate clearly whether the theme is based on Bootstrap, and for which version(s).

Here is a screenshot from the [Canvas theme](http://themeforest.net/item/canvas-the-multipurpose-html5-template/9228123) as an example:

![](/assets/images/aspnet-developers-index-to-bootstrap/themeforest.png)

You can also just search for the term "bootstrap" in the HTML (Site Templates) category: 

http://themeforest.net/category/site-templates?term=bootstrap    

Another option is to look at Themeforest's sister site [Codecanyon](http://codecanyon.net) in the [Bootstrap Skins category](http://codecanyon.net/category/skins/bootstrap). These are not full blown themes like with Themeforest, but more "lightweight" skins, similar to what Bootswatch have got.  

## Build faster

The last section I want to cover is a set of tools which will allow you to develop Bootstrap websites faster.

### Visual Editors

Both [Jetstrap](https://jetstrap.com/) and [Brix](http://brix.io/) are visual editors which allow you to create website layouts by dragging Bootstrap components onto a canvas. Once you are happy with the design it is a simple matter of exporting the HTML. Brix even has a number of pre-made templates which you can choose from to customise.

### Code snippets

If find [Bootsnipp](http://bootsnipp.com/) a very useful website when I am looking for pre-build components. Looking for some examples of login forms? Simply [search for the term "login"](http://bootsnipp.com/search?q=login) and you will find a number of layouts for login forms for which you can easily copy and paste the HTML into your own application. 

### Chameleon Forms
[Chameleon Forms](https://github.com/MRCollective/ChameleonForms) is a .NET library which aims to make building forms in ASP.NET MVC a much more powerful and easier experience. It allows you to build forms in a declarative way and get rid of a lot of the boilerplate HTML code you normally have to add when creating forms, and comes with support for Bootstrap 3 built it.

## Conclusion

I hope this post introduced you to a few resources which you were not familiar with before. Feel free to share your favourite Bootstrap resources in the comments below.