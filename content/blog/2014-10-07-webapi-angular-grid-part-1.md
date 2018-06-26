---
date: 2014-10-07T00:00:00Z
description: |
  Demonstrates how to build a grid with sorting and paging using ASP.NET Web API, AngularJS, Restangular and ng-Table. This is Part 1 which covers the basic setup.
tags:
- aspnet
- aspnet web api
- angularjs
title: 'Building an interactive grid with ASP.NET Web API and AngularJS: The basic
  setup (Part 1)'
url: /blog/webapi-angular-grid-part-1/
---

## Introduction
I am currently involved in a project in which we use a combination of ASP.NET MVC and AngularJS powered by a ASP.NET Web API backend. For one of the pages we need to display tabular data in a grid where the user can filter, sort and page through the data. The Web API backend is a pretty standard REST based API and for communication between the API and the Angular frontend I have been using [Restangular](https://github.com/mgonto/restangular). Looking on the Angular Modules website it [seemed as though most people favour](http://ngmodules.org/tags/table) the use of [ngTable](http://bazalt-cms.com/ng-table/) as an AngularJS table plugin. I did a small proof of concept and it seemed to work for my purposes, so I decided to use it for this project.

I learned quite a few things in building this seemingly simple little feature and I thought it would make for a nice little tutorial on how to put all these technologies together. This is the first in a series of blog posts in which I will walk you through the process of creating a filterable, sortable grid with paging using the combination of ASP.NET Web API, AngularJS, Restangular and ngTable.

## Creating the basic ASP.NET MVC project
To start off we create a normal ASP.NET Web Project in Visual Studio

![](/assets/images/webapi-angular-grid/new-project.png)

Select the template for an MVC project and make sure you also select the `Web API` check box. Click OK.  

![](/assets/images/webapi-angular-grid/new-aspnet-project.png)

Once the project has been created I find it good practice to ensure that all the Nuget packages are updated to the latest versions, so go ahead and do that if you want to. 

## Install the required libraries
Next up we need to install all of the Javascript libraries we will need. You can do that by either installing the Nuget packages for these (if available) or alternatively download them from the web and copy the relevant javascript files into you `scripts` folder. In the case of ngTable there are also some CSS stylesheets you will need to copy into your `Content` folder.

Let's run over each of these quickly:

### Angular
For Angular there is an up to date Nuget package, so search for the Nuget package with the ID `angularjs` and install it.

![](/assets/images/webapi-angular-grid/angular-nuget.png)

The Nuget package will install a whole bunch of javascript files into the `Scripts` folder of your project.

### Restangular
The Nuget package for Restangular was outdated so you can go to the Github releases for Restangular (https://github.com/mgonto/restangular/releases) and download the latest version. Extract the .ZIP file and copy the files `restangular.js` and `restangular.min.js` to your `Scripts` folder.

### Underscore
Restangular requires either Lodash or Underscore to be installed. Go to the Underscore page at http://documentcloud.github.io/underscore/ and download the latest minified version. Copy the file (`underscore-min.js`) to your `Scripts` folder. You can also optionally download the source map file (`underscore-min.map`) and copy that to the `Scripts` folder. 

### ngTable
Go to to ngtable website (http://bazalt-cms.com/ng-table/), download the latest version and extract the .ZIP file. Copy the files `ng-table.js`, `ng-table.min.js` and `ng-table.map` to your `Scripts` folder. Copy the stylesheet files (`ng-table.css`, `ng-table.min.css` and `ng-table.less`) to your `Content` folder.

## Update the Bundles
We need to update the CSS bundle to include the ngTable CSS file, so open the file `App_Start/BundleConfig.cs` and edit the definition of your CSS bundle to also include the file `ng-table.min.css`.

```csharp
bundles.Add(new StyleBundle("~/Content/css").Include(
            "~/Content/bootstrap.css",
            "~/Content/ng-table.min.css",
            "~/Content/site.css"));
```

Also create a new script bundle called `app` to include all of the Javascript files we will need for the AngularJS application:

```csharp
bundles.Add(new ScriptBundle("~/bundles/app").Include(
    "~/Scripts/angular.min.js",
    "~/Scripts/underscore-min.js",
    "~/Scripts/restangular.js",
    "~/Scripts/ng-table.min.js"
    ));
```

## Create the AngularJS application
For the AngularJS application I create a separate folder called `App`. Inside this folder I create a file called `app.js` which declares my AngularJS application. Make sure to also add `restangular` and `ngTable` as dependencies.

```js
(function () {
    'use strict';

    angular.module('app', [
        // Angular modules 

        // Custom modules 

        // 3rd Party Modules
        'restangular',
        'ngTable'
    ]);
})();
```

Next up create the AngularJS controller. For this example we will demonstrate how to bind the grid to a list of customers, so I create a file called `customers.controller.js` in my `App` folder and inside of this file I define the controller called `CustomersController`.

```js
(function () {
    'use strict';

    angular
        .module('app')
        .controller('CustomersController', CustomersController);

    CustomersController.$inject = ['$location']; 

    function CustomersController($location) {
        /* jshint validthis:true */
        var vm = this;
        vm.title = 'CustomersController';

        activate();

        function activate() { }
    }
})();
```

At this point I just want to make a quick mention of the fact that I used the SideWaffle templates to create my AngularJS app and controller. If you have not done so before have a look at the SideWaffle website (http://sidewaffle.com/) as it adds a whole bunch of really nice project and file templates to Visual Studio. In my case I used the `AngularJs Module` template to create the AngularJS application and the `AngularJs Controller` template to create the Customers Controller.  

## Add the files to your script bundle
We need to update our application script bundle to add the 2 files we just created, so open the `BundleConfig` class again and update the bundle definition inside the `RegisterBundles()` method.

```csharp
bundles.Add(new ScriptBundle("~/bundles/app").Include(
    "~/Scripts/angular.min.js",
    "~/Scripts/underscore-min.js",
    "~/Scripts/restangular.js",
    "~/Scripts/ng-table.min.js",
    "~/App/app.js",
    "~/App/customers.controller.js"
    ));
```

I also suggest that you disable optimization in your bundles, i.e. no combining of scripts / stylesheets or minifications. It is going to make your life a whole lot easier if you need to debug later to find issues, so locate the line `BundleTable.EnableOptimizations = true` in your `BundleConfig` class and comment it out:

```csharp
// Set EnableOptimizations to false for debugging. For more information,
// visit http://go.microsoft.com/fwlink/?LinkId=301862
//BundleTable.EnableOptimizations = true;
```  

## Update the HTML
We need to add the `app` bundle to our layout file, so open your `_Layout.cshtml` file and add the `app` bundle after the two existing bundles:

```html
@Scripts.Render("~/bundles/jquery")
@Scripts.Render("~/bundles/bootstrap")
@Scripts.Render("~/bundles/app")
```

We also need to hook up the Angular application so edit the `<body>` tag of your layout file to add the `ng-app` directive:

```html
<body ng-app="app">
    <div class="container body-content">
        @RenderBody()
    </div>

    ...
</body>
```

The last bit is to declare the `ng-controller` in our home page. Open the `Views\Home\Index.cshtml` file and delete all of the existing content and add the HTML snippet below:

```html
<div ng-controller="CustomersController as vm">
    <h1>{{ vm.title }}</h1>    
</div>
```

You will notice in the HTML snippet that I have added the expression `{{ vm.title }}` to my `<H1>` element. This is just a temporary checkpoint so we can run the application at this point and make sure that everything is hooked up correctly. So go ahead and run the application.

![](/assets/images/webapi-angular-grid/part-1-checkpoint.png)

If you see the text "CustomersController" like in the screenshot above it means all the scripts are loading correctly and the Angular application and controller has run without any errors. 

If you see the text "{{ vm.title }}" it means the expression was not successfully evaluated and something is not hooked up correctly. If this is the case you will need to use your browser developer tools to ensure that all scripts loaded correctly and if there are any Javascript errors you will need to see what they are and correct them. Unfortunately I am not available for remote debugging, so you're on your own if this is the case ;)

## Conclusion
In this blog post I created the basic framework for the application and ensured that all the infrastructure is in place. Next time will will create an API to get the list of customers and hook it up to the grid.

In [Part 2](http://www.jerriepelser.com/blog/webapi-angular-grid-part-2), we will be adding a basic grid. 
