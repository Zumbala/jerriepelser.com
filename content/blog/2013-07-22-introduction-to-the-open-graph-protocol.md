---
date: 2013-07-22T00:00:00Z
description: |
  A brief introduction to the Open Graph protocol which you can use to add context to your HTML pages which is understood by Facebook and others.
tags:
- facebook
- open graph
- social media
title: Introduction to the Open Graph Protocol
url: /blog/introduction-to-the-open-graph-protocol/
---

## Background

Web pages have been designed for humans to read and consume.  There has however been a need for probably as long  as the World Wide Web has existed to be able to understand the content of a web page and for computers to be able to ascertain the meaning of the content (or data) contained inside an HTML document.  Along the way there has been various efforts to bring more structure to the actual data contained in web pages.  These days with HTML5 gaining popularity you hear a lot about [semantic markup](https://en.wikipedia.org/wiki/Semantic_HTML), which is essentially yet another effort to make the semantics (or the meaning) of a web page clear.

Companies like Google and Facebook have a need to understand what the content of a web page is about.  For Google this is important because if they can understand what the content of a web page relates to, they are able to serve more relevant ads and bring back more relevant search results.  For Facebook it is important because they want to build up their [Open Graph](https://developers.facebook.com/docs/opengraph/) of _stories_ about you, and if they can understand the content behind the web pages you are liking or sharing better they are able to understand you better and that makes you more valuable to them.

Both of these companies try and achieve this by getting web site owners to embed meta data inside their web pages which will allow them to get a better understanding of the "meaning" of a web page.  Google favours [Rich Snippets](https://support.google.com/webmasters/answer/99170?hl=en), while Facebook is going for the [Open Graph protocol](http://ogp.me/).

I came across both these in the development of my social media sharing application [One Love](http://www.oneloveapp.com), and have since come to understand both of these a little bit better.

In this article I will give a quick overview of what the Open Graph protocol is.

## What is The Open Graph Protocol?

I will quote directly from the [Open Graph protocol website](http://ogp.me/):

> The [Open Graph protocol](http://ogp.me/) enables any web page to become a rich object in a social graph. For instance, this is used on Facebook to allow any web page to have the same functionality as any other object on Facebook.

The Open Graph protocol is based on [RDFa](http://en.wikipedia.org/wiki/RDFa), which is a way to extend HTML (or XML) by adding attributes to elements which provides certain metadata about the particular document.  This metadata is embedded inside a web page by adding HTML <meta> tags inside the <head> tag of a page.  Below is an example (taken from the Open Graph protocol website) of an HTML document containing Open Graph metadata tags.

``` html
<html prefix="og: http://ogp.me/ns#">
<head>
<title>The Rock (1996)</title>
<meta property="og:title" content="The Rock" />
<meta property="og:type" content="video.movie" />
<meta property="og:url" content="http://www.imdb.com/title/tt0117500/" />
<meta property="og:image" content="http://ia.media-imdb.com/assets/images/rock.jpg" />
...
</head>
...
</html>
```

In the example above we can see that the Title, URL and Image attributes are specified, as well as a type metadata attribute which can tell Facebook exactly what sort of data it is that the web page contains.  In this can they can essentially see that you are sharing a web page about a movie called "The Rock".

The four attributes above are the basic required attributes defined in the Open Graph protocol, but there is a lot more metadata which can be attached to a web page which can define a lot more detail.  For a movie for example it can contain information such as the running time, release date, actors, directors and writers.  For a web page about an episode of a specific television series it can contain the same example as above, but also link back to to actual television show.

As you can see, the web page documents you are looking at can contain a rich metadata underneath.  This metadata can also contain links to other objects, which allow for building up a rich graph of objects with their relationships.  In the example of an actor being defined for a movie, it basically contains a link (i.e. a URL) to the page for that actor which in turns contains a whole lot of extra information regarding the actor.

For more detailed information I suggest you [have a look at the Open Graph protocol website](http://ogp.me/) for yourself.

## Where can I see it in action?

The easiest way to see the Open Graph protocol in action in your daily life is to simply go to Facebook and share a link.  In the example below I simply pasted the link to The Rock ([http://www.imdb.com/title/tt0117500/](http://www.imdb.com/title/tt0117500/)) inside the Facebook status update box and Facebook was able to retrieve the title, description and an image of that link.

![](/assets/images/2013/07/Capture.png)

So where did Facebook get that information?  They got it from the Open Graph protocol data inside the web page of course.  Below is a snippet of the relevant HTML source code from that specific web page:

``` html
<meta property="og:image" content="http://ia.media-imdb.com/assets/images/M/MV5BMTM3MTczOTM1OF5BMl5BanBnXkFtZTYwMjc1NDA5._V1_SY317_CR4,0,214,317_.jpg" />
<meta property='og:type" content="video.movie" />
<meta property='og:title" content="The Rock (1996)" />
<meta property="og:description" content="Directed by Michael Bay.  With Sean Connery, Nicolas Cage, Ed Harris, John Spencer. A renegade general and his group of U.S. Marines take over Alcatraz and threaten San Francisco Bay with biological weapons. A chemical weapons specialist and the only man to have ever escaped from the Rock attempt to prevent chaos." />
```

Another way to look at the Open Graph data is to go to the [Facebook Debug tool](https://developers.facebook.com/tools/debug) and enter the URL of a web page and click the **Debug** button.  It will display the Open Graph data about the website as well as some other related information.  Using the Debug Tool you will notice that when Open Graph attributes are missing from a web page, Facebook will infer the information by scraping it from other elements in the web page.  To get the title it will for example look at the the `<title>` element of the page.

## How do I enable it on my website?

Facebook would of course want all of us to play along nicely and enable these Open Graph attributes inside our own websites, so how do you do it?  Well that depends completely on your website.

My guess is that about all of the popular content management systems has this already built in, or will have some sort of plugin available which enables it.  I was surprised to find that my own blog already had this enabled even though I had no recollection of ever enabling it or installing a plugin which enables it.  After a quick search I found it was the [Yoast Wordpress SEO plugin](http://yoast.com/wordpress/seo/) which I installed which enabled it.

If you are using static HTML pages you will need to add these tags manually.  If you have a custom developed website using some technologies like ASP.NET, Ruby on Rails or any of the other server side or client side frameworks out there, you will need to write some code (or ask your developers to write some code) to add these metadata attributes to your website.

## How do I consume it?

Of course it is not just Facebook who can use the Open Graph data which is embedded inside a web page.  In the next blog post I will give an example of how you can extract Open Graph meta data from a web page using C# - this is a programming blog after all.