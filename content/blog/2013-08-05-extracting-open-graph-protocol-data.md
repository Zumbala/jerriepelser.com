---
date: 2013-08-05T00:00:00Z
description: |
  Demonstrates how you can download Open Graph data from a web page using the AngleSharp library.
tags:
- anglesharp
- open graph
- social media
title: Extracting Open Graph Protocol Data
url: /blog/extracting-open-graph-protocol-data/
---

In my [previous](/blog/introduction-to-the-open-graph-protocol/) blog post I gave an overview of the Open Graph Protocol and gave a few examples of how it is being used in web pages.  My exploration of the Open Graph protocol started off when I wanted to add the ability for users to easily share links using [One Love](http://www.oneloveapp.com/), the application which I am developing.

## Downloading and extracting the Open Graph Data

In this section we will be developing a very basic WPF application which downloads a web page using HttpClient, extracts the Open Graph data from the HTML and display it to the user.  For the purposes of extracting the information from the HTML I will use [AngleSharp](http://www.nuget.org/packages/AngleSharp/) which was developed by Florian Rappl.  I prefer this library as it is a Portable Class Library so it suited my specific needs when developing [One Love](http://www.oneloveapp.com/) much better, but I understand that [HTML Agility Pack](http://www.nuget.org/packages/HtmlAgilityPack/) is also a very popular choice for this kind of work.

We start of with downloading and extracting the web page using HttpClient.  The downloaded HTML is then passed to the AngleSharp DocumentBuilder class which will parse it and allow us to query it for Open Graph attributes.

``` csharp
// Download the web page
HttpClient httpClient = new HttpClient();
string html = await httpClient.GetStringAsync(uri);

// Load the HTML Document
HTMLDocument document = DocumentBuilder.Html(html);
```

We will use the QuerySelectorAll() method of AngleSharp which allows us to query for specific DOM elements using [CSS Selectors](http://www.w3schools.com/cssref/css_selectors.asp).   Remember in the previous article we stated that the Open Graph title metadata will look as follows:

``` html
<meta property="og:title" content="The Rock (1996)" />
```

To extract this metadata we will query for all `<meta>` elements which has an attribute named `property` with the value `og:title`.  The CSS selector for this is as follows:

`meta[property="og:title"]`

The code below uses AngleSharp to query for the first HTML element which satisfies the query, and if we find it we check for the `content` attribute of that element to get the actual title.

``` csharp
var titleElement = document.QuerySelectorAll("meta[property=\"og:title\"]")
                        .FirstOrDefault();
if (titleElement != null)
    Title = titleElement.GetAttribute("content");
```

A lot of web pages however does not contain Open Graph protocol data, so in that case we will fall back to the `<title>` element of the page.  The expanded code is below:

``` csharp
var titleElement = document.QuerySelectorAll("meta[property=\"og:title\"]")
                        .FirstOrDefault();
if (titleElement != null)
    Title = titleElement.GetAttribute("content");
else
{
    // Try and get from the <TITLE> element
    titleElement = document.QuerySelectorAll("title")
                            .FirstOrDefault();
    if (titleElement != null)
        Title = titleElement.TextContent;
}
```

We then continue in a similar fashion to extract the description and images Open Graph metadata.  In the case where the Open Graph description metadata does not exist, we simple attempt to fall back to the **<meta description="">** HTML element.

``` csharp
// Try and get the description from OpenGraph data
var descriptionElement = document.QuerySelectorAll("meta[property=\"og:description\"]")
                        .FirstOrDefault();
if (descriptionElement != null)
    Description = descriptionElement.GetAttribute("content");
else
{
    descriptionElement = document.QuerySelectorAll("meta[name=\"Description\"]")
                            .FirstOrDefault();
    if (descriptionElement != null)
        Description = descriptionElement.GetAttribute("content");
}

// Try and get the images from OpenGraph data
var imageElements = document.QuerySelectorAll("meta[property=\"og:image\"]");
foreach (var imageElement in imageElements)
{
    Uri imageUri = null;
    string imageUrl = imageElement.GetAttribute("content");
    if (imageUrl != null && Uri.TryCreate(imageUrl, UriKind.Absolute, out imageUri))
        Images.Add(imageUri);
}
```

Below is a screenshot of the final running application:

![](/assets/images/2013/08/2013-08-04-11-57-53-PM1.png)

You are welcome to clone the Github repository for this blog post (details below) and start experimenting with using Open Graph protocol data yourself.

## Resources

For more detailed documentation on AngleSharp I suggest you read Florian's [article about AngleSharp on Code Project](http://www.codeproject.com/Articles/609053/AngleSharp).  Also have a look at the [AngleSharp repository on Github](https://github.com/FlorianRappl/AngleSharp) which contains a sample project.

For the modern UI look of the demo WPF application I used [MahApps.Metro](http://mahapps.com/MahApps.Metro/).
