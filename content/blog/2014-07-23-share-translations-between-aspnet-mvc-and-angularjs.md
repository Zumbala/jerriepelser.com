---
date: 2014-07-23T00:00:00Z
description: |
  The ASP.NET MVC (or Web API) and AngularJS parts of your apps share the same translations. This shows how you can share those using a single source for translations.
tags:
- aspnet mvc
- angularjs
- localization
title: Share translations between ASP.NET MVC and AngularJS
url: /blog/share-translations-between-aspnet-mvc-and-angularjs/
---

I am currently helping a friend on a project which involves a mobile application running on iOS and Android, with a backend API and supporting administration website which is developed in ASP.NET MVC and Web API. The application is targeted at an international audience and one of the requirements were that the application should support multiple languages.

I am responsible for the development of all the web related software and for the administration website I am using AngularJS with ASP.NET Web API, and also a few pages which are just plain old ASP.NET MVC.

## Handling translation in AngularJS
I am pretty much familiar with handling localization of applications in ASP.NET MVC so I set off trying to find something which could handle doing translations for me on the AngularJS side. After looking at a few options I settled on [angular-translate](http://angular-translate.github.io/).

The documentation is very good and installing it is straight forward. A Nuget package [is available](https://www.nuget.org/packages/AngularTranslate), but it is not current so you may want to rather [download the latest release directly from GitHub](https://github.com/angular-translate/angular-translate/releases).

Angular translate supports various methods in which to supply translations for the different languages. You can specify the translations directly in your app, or you can load it asynchronously from an external source [using one of the various loaders](http://angular-translate.github.io/docs/#/guide/12_asynchronous-loading). 

I decided on using the the `staticFilesLoader` as the whole process seemed fairly straight forward and easy to get going. For the `staticFilesLoader` one simply maintains a set of translation files in JSON for each supported language. The loader does the job of asynchronously loading the correct translation file when needed. 

It was however not long before I ran into a few concerns. The first concern was keeping the various translation files in sync, and my second (and biggest) concern was the fact that even though I had solved the issue of handling translations for the AngularJS side of the application, I was still maintaining a completely separate set of translation files for the ASP.NET MVC and Web API side of this which was using normal .NET `.resx` files.

## Sharing the same translation files
One of the other ways angular-translate can handle loading of translations is by loading it from a URL using the `urlLoader`. For configuring the `urlLoader` you simply need to supply a URL from where the translations will be loaded. When Angular Translate requires a translation set for a specific language, it will make a request to that URL and pass in a `lang` query string parameter which specifies the language requested. 

A request for the English language translations will for example be made to the URL

```bash
/api/translations?lang=en
``` 

The response from the URL must be a JSON object containing all the translations as key-value pairs, for example:

```json
{
	"FirstName" : "First name",
	"LastName" : "Last name"
}
```

This seemed like a good solution to me as all I needed was to create a resource on my Web API which will read the translations from my `.resx` file and serve them up in the correct JSON format.

The resulting Web API method ended up looking like this:

```csharp
public class TranslationsController : ApiController
{
    public IHttpActionResult Get(string lang)
    {
        var resourceObject = new JObject();

        var resourceSet = Resources.Resources.ResourceManager.GetResourceSet(new CultureInfo(lang), true, true);
        IDictionaryEnumerator enumerator = resourceSet.GetEnumerator();
        while (enumerator.MoveNext())
        {
            resourceObject.Add(enumerator.Key.ToString(), enumerator.Value.ToString());
        }

        return Ok(resourceObject);
    }
}
```

In the method above I declare an instance of the JSON.NET `JObject` class which will contain all the key-value pairs for the translations. I then get the resource set for the requested culture and simply iterate through all the resources and add them to the instance of the `JObject`. Finally I simply return the instance of `JObject` which will send the serialized JSON back.

The next step was to configure my AngularJS application to use Angular Translate. Below is the definition of my AngularJS application:

```javascript
(function () {
    var app = angular.module('app', ['ngCookies', 'pascalprecht.translate']);

    app.config(['$translateProvider',
        function ($translateProvider) {
            $translateProvider.useUrlLoader('/api/translations');
        }]);

})();
```

I also want the user to be able to switch between the languages, so I created a controller which gives handles that:

```javascript
angular.module('app').controller('DemoController', [
    '$scope', '$cookies', '$translate', function($scope, $cookies, $translate) {
        $scope.setLanguage = setLanguage;

        function setLanguage(lang) {
            $cookies.__APPLICATION_LANGUAGE = lang;
            $translate.use(lang);
        }

        function init() {
            var lang = $cookies.__APPLICATION_LANGUAGE || 'en';
            setLanguage(lang);
        }

        init();
    }
]);
``` 

The functionality of this controller is quite simple. I have defined a `setLanguage` method on the controller which sets a cookie with the correct value for the current language and then notifies Angular Translate to use that language using the `$translate.use()` method. Upon initialization of the controller I also try and determine the current language by inspecting the value of the cookie, and if it is not set I default it to `en`.

The reason for setting the cookie is because my ASP.NET MVC backend will pick up this value and set the correct culture for the current thread at the start of each request.  So in the Global.asax.cs I add the following method to my MvcApplication class:

```csharp
protected void Application_BeginRequest()
{
    string language = "en";

    var cookie = Request.Cookies.Get("__APPLICATION_LANGUAGE");
    if (cookie != null)
        language = cookie.Value;

    Thread.CurrentThread.CurrentCulture = new CultureInfo(language);
    Thread.CurrentThread.CurrentUICulture = new CultureInfo(language);
}
```

## Testing it out
So my very basic test application has a view  which displays a greeting in the current language:

```html
<div ng-controller="DemoController">
    <p class="lead">This is translated using MVC:</p>
    <h1>@SharedTranslationsAspnetAngular.Resources.Resources.WhatsYourName</h1>
    <p class="lead">This is translated using angular-translate:</p>
    <h1 translate>WhatsYourName</h1>
    <p class="lead">Change language to:</p>
    <a class="btn btn-default" ng-click="setLanguage('en')">English</a>
    <a class="btn btn-default" ng-click="setLanguage('es')">Spanish</a>
    <a class="btn btn-default" ng-click="setLanguage('fr')">French</a>
</div>
``` 

The first greeting is displayed using the normal ASP.NET MVC method to output the value of a string resource:

```html
<h1>@SharedTranslationsAspnetAngular.Resources.Resources.WhatsYourName</h1>
```

For the second greeting I use the Angular Translate `translate` directive to do the translation:

```html
<h1 translate>WhatsYourName</h1>
```

When you run the application you are initially greeted by a screen which displays a greeting in English:

![](/assets/images/share-translations-between-aspnet-mvc-and-angularjs/initial-view.png)

Clicking on the button labeled "Spanish" displays the greeting in Spanish:

![](/assets/images/share-translations-between-aspnet-mvc-and-angularjs/angular-translated-view.png)

You will notice that only the Angular Translate greeting has switched to Spanish. This is because Angular Translate has made an AJAX request back to the server to fetch the Spanish translations and applied them to the elements with the `translate` directive. The ASP.NET MVC greeting is however still displayed in English as we will need to do a full page refresh to ASP.NET MVC can serve the Spanish translated view. 

Pressing F5 so that the browser can do a full page refresh back from the server now displays both greetings in Spanish:

![](/assets/images/share-translations-between-aspnet-mvc-and-angularjs/full-translated-view.png)

## Translation tools
I mentioned before that one of my gripes with defining the translations in the JSON files was trying to keep the different translations files in sync. So when you add a new translation key to the JSON file containing the English translations for example, you must remember to add that key and the associated translation to all the other JSON files. All the JSON files are also edited separately. The process is laborious and error prone.

Since we are now maintaining the translations on `.resx` files there are a number of tools available which also makes the process of maintaining the different translation files much easier. One of the ones which I found quite effective, and is available for free is called [Zeta Resource Editor](http://www.zeta-resource-editor.com/index.html).

![](/assets/images/share-translations-between-aspnet-mvc-and-angularjs/zeta.png)

Zeta allows you to view all you translations in a single view which gives you a quick overview of which translations are missing. You can also change the name of a translation key quite easily and have it automatically applied to all the translation files.
   