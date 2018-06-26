---
date: 2013-08-12T00:00:00Z
description: |
  When there are no official APIs to integrate with an external system you can always fall back on the time tested method of screen scraping. We take a look at how you can do this.
tags:
- html
- web scraping
title: 'Plan B: Data scraping'
url: /blog/plan-b-data-scraping/
---

I enjoy listening to podcasts.  I will listen mostly to technical or entrepreneurial podcasts, but I also like to listen to podcast about subjects which are completely different.  One of my favourite podcasts is [NPR Planet Money](http://www.npr.org/blogs/money/).  I enjoy it first and foremost because of the wonderful story telling in every episode, and also just because the usually cover such interesting topics.  One episode a while back stands out to me as a techie - [Episode 396: A Father Of High-Speed Trading Thinks We Should Slow Down](http://www.npr.org/blogs/money/2013/06/14/191668134/episode-396-a-father-of-high-speed-trading-thinks-we-should-slow-down).

It is about the history of Thomas Peterffy who is one of the key figures in the development of electronic trading of  securities. The episode describes how he developed automated ways to get into the data feeds of Quotron machines in the late 70's/early 80's and then in the late 80's when the Nasdaq was opened he figured out a way once again hook into the Nasdaq data feeds, feed the data into his computer which did some number crunching and then automatically place orders.  My summary does not do his story justice.  You really need to go and [listen to the episode](http://www.npr.org/blogs/money/2013/06/14/191668134/episode-396-a-father-of-high-speed-trading-thinks-we-should-slow-down).

At the end of last year I sold all my belongings and I am now travelling the world.  I tend to travel very slowly and stay at a place for weeks or months at a time.  I also meet a lot of people on my travels who do the same as me (i.e. travel and have some way to make money on the internet) and it always fascinates me to hear of the clever ways in which people make money.  In Hue, Vietnam I met a German called Michael.  He is clever with computers and his friend is clever with formulas.  Very similar to the story of Thomas Peterffy, they have figured a way to do automated betting on horse races.  They basically scrape data from web pages, feed it into their system which does some number crunching and then based on that automatically places bets on horse races by doing some clever automation of web pages.

The purpose of these stories?  At some point in your life as a programmer you will probably run into a scenario where there is no standard documented or supported way to do something.  Your boss will want it done and you will need to find a Plan B to make it happen.  It may involve something complicated like splicing of cables and tapping into data streams, or something more simple like scraping data from a web page.  I love that about our industry - the fact that every now and again our creativity is challenged and that we need to come up with an innovative way in which to solve a problem.

In the rest of this blog post I will show you a very simple way to do some basic web scraping.  There is nothing ground breaking or revolutionary about this.  It is simply yet another one of the hundreds of little techniques which us developers need in our tool chest at some stage in our careers. When that time comes maybe this blog post will help you out.

## Let's write some code

In a [previous blog post](/blog/extracting-open-graph-protocol-data/) I described how to use [CSS selectors](http://www.w3schools.com/cssref/css_selectors.asp) to extract Open Graph data from an HTML document.  CSS selectors are used in Cascading Style Sheets, to determine which style rules to apply to a document.  Because of their pattern matching ability they can also be used to describe a pattern for DOM elements to select from a document.  This ability is for example [used in jQuery](http://api.jquery.com/category/selectors/) to select elements in a document which can then be manipulated further.

For the purpose of this blog post I will be using [HTML Agility Pack](http://htmlagilitypack.codeplex.com/) to parse the HTML document.  I also use [Fizzler](https://github.com/yetanotherchris/fizzler/) which extends HTML Agility Pack to allow me to select HTML elements with CSS selectors.

Seeing as I like traveling I will demonstrate how to extract the list of specials from the Geckos Adventures website which is located at [http://www.geckosadventures.com/browse/specials](http://www.geckosadventures.com/browse/specials).  The first bit is to identify which CSS selectors to use to extract the correct DOM elements.  All modern browsers will have some sort of page inspector which you can use to inspect DOM elements.

![](/assets/images/2013/08/2013-08-12-07-00-27-PM.png)

As you can see in the screenshot above the list of specials are contained in **DIV** elements with the class **.views-row**.  To select each record we want to display we can therefore use the CSS selector **div.views-row** to select the list of records.  The basic code for this looks as follows:

``` csharp
using (HttpClient httpClient = new HttpClient())
{
    string html = await httpClient.GetStringAsync(uri);

    HtmlDocument document = new HtmlDocument();
    document.LoadHtml(html);

    // Select the individual items
    var tripNodes = document.DocumentNode.QuerySelectorAll("div.views-row");
}
```

Once we have the list for DIV elements we can process each of them further to extract the image, description and price.  The title of the trip is contained inside an H4 element.  It is the only H4 element inside the DIV so we can simply specify the CSS selector H4 to select the title.  If there were more than one H4 inside the DIV you would have had to specify a more specific CSS selector to ensure you get the correct element. Is is important for you to ensure that you are specific enough with your CSS selectors to only select the elements in which you are interested. Following a similar fashion we identify the CSS selectors to select the other elements.

``` csharp
string title = WebUtility.HtmlDecode(node.QuerySelector("h4")
    .InnerText);
string imageUrl = node.QuerySelector(".search-item-left .search-item-hero a img")
    .GetAttributeValue("src", null);
string tripUrl = "http://www.geckosadventures.com" + node.QuerySelector("a.search-item-view-trip")
    .GetAttributeValue("href", null);
string salesPriceText = node.QuerySelector(".search-item-left .pg_tripcode")
    .InnerText;
```

Below is a screenshot of the running application

![](/assets/images/2013/08/2013-08-12-08-08-24-PM.png)
