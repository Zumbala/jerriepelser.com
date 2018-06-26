---
date: 2015-09-22T00:00:00Z
description: |
  I demonstrate how you can create your own very lightweight API wrapper when you don't want to use a large external library to access with external APIs, or when no official wrappers exists for an API.
tags:
- asp.net
- oauth
- API
- refit
title: Creating a lightweight API wrapper with Refit
url: /blog/create-lightweight-wrappers-with-refit/
---

## Introduction

With the explosion of web APIs it becomes much more common for applications to integrate in some fashion with external APIs. These APIs are typically RESTful APIs, and normally there will be an official wrapper available for a wide range of programming languages and platforms to allow other developers to more easily integrate with the API.

Sometimes however this may not be the case. It may be that the service in question simply do not see providing a .NET API wrapper as a priority, or that the official wrapper may be targeting the full .NET Framework, and you may be looking for something that you can use from WinRT, Xamarin or PCL libraries in general.

It may also be that you simply do not want to include a large external dependency in your application, or you may actually need an API wrapper for one of your own APIs.

In all of these cases you may be required (or tempted) to write you own wrappers. Writing API wrappers can however be a tedious process.

In these sort of situations I revert to using Refit, which is an open-source PCL library developed by Paul Betts, and it works across just about every flavour of the .NET framework - Xamarin, WinRT, Windows Phone 8, UPW, etc.

## Defining the interface for an API

In this blog post I will be looking at creating a wrapper around the GitHub API method which [renders markdown as HTML](https://developer.github.com/v3/markdown/). I suggest you quickly look at the documentation for that method to see it is all about.

The first thing you need to do is to add Refit to your application. Install it with Nuget:

``` text
Install-Package refit
```

After the packages has been added we will need to create the definition for the API and the methods we are going to call. For the API we simply define an interface, and then define the API methods we want to call as methods on the interface:

``` csharp
[Headers("User-Agent: Refit Demo App")]
public interface IGitHubApi
{
	[Post("/markdown")]
	Task<string> RenderAsMarkdown([Body] RenderRequest request);
}
```

A few things to note:
1. The method `RenderAsMarkdown` returns a `Task` of string. Refit requires that the return type always be a Task, as it operate asynchronously. So if the actual API call returns a `string` like in my case, then the return type will be `Task<string>`.
2. To specify the actual endpoint for the method, you decorate the method with a `[Get]`, `[Post]`, `[Put]`, `[Patch]` or `[Delete]` attribute, passing in the endpoint as parameter. In our case, to call the API method you will do an HTTP POST to `https://api.github.com/markdown`, so I specify the `[Post("/markdown")]` attribute. Notice that I do not specify the full URL for the endpoint, but only the `/markdown` part. I will specify the base address for the API (i.e. `https://api.github.com`) later on.
3. For the parameter of the method, I pass in an instance of the `RenderRequest` class. I also decorate the parameter with the `[Body]` attribute, which indicates to Refit that it should serialize the object to JSON and pass it in the body of the HTTP request.
4. I specified a `[Headers]` attribute on the interface definition. The reason is because the GitHub API [requires that a User-Agent header always be specified](https://developer.github.com/v3/#user-agent-required) with all API calls. By adding this attribute, Refit will always pass in the headers specified in that attribute along with every request.

As you can see in the documentation for the API method, the method expects a JSON document to be passed in the body which looks something like the following:

``` json
{
  "text": "Hello world github/linguist#1 **cool**, and #1!",
  "mode": "gfm",
  "context": "github/gollum"
}
```

So I create my `RenderRequest` class to represent that structure:

``` csharp
public class RenderRequest
{
    [JsonProperty("text")]
    public string Text { get; set; }

    [JsonProperty("mode")]
    public string Mode { get; set; }

    [JsonProperty("context")]
    public string Context { get; set; }
}
```

To make sure that the property names are not serialized as Pascal-case strings, I specify the correct, camel-case name of the property by using the `[JsonProperty]` attribute from JSON.NET - Refit uses JSON.NET under the covers for serialization and de-serialization of JSON.

## Consuming an API

Consuming the API is quite simple. Simply create an instance of the API using the `RestService.For<>` method, passing in the base URL for the API as parameter and then call the API methods you defined. So in our case the code to call the API looks as follows:

```csharp
var gitHubApi = RestService.For<IGitHubApi>("https://api.github.com");

var request = new RenderRequest
{
    Text = "Hello **world**, this is sample markdown [with a URL](http://www.somewhere.com)",
    Mode = "markdown"
};

var response = await gitHubApi.RenderAsMarkdown(request);
```

This takes the markdown I passed in, and converts it to the following HTML. 

``` html 
<p>Hello <strong>world</strong>, this is sample markdown <a href="http://www.somewhere.com">with a URL</a></p>
```
 
## More advanced example

The example I used in this blog post was quite simple. I have also created another, more advanced example for my [AspnetCasts YouTube channel](http://www.aspnetcasts.com/), which you can view below. This example employs a few more advanced techniques such as specifying a JSON formatter, returning a JSON document and passing along OAuth credentials.

Enjoy :)

<iframe width="560" height="315" src="https://www.youtube.com/embed/Myv7Hb90s5A" frameborder="0" allowfullscreen></iframe>

## Conclusion

In this blog post I demonstrated how you can very easily and quickly create a C# wrapper for just about any REST API by using Refit. 