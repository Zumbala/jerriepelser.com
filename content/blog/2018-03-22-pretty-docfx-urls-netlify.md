---
title: Generate "pretty" URLs for DocFX websites with Netlify
description: |
  By default, DocFX generated websites contain the .html file extension in the URL. You can use Netlify's asset optimization features to clean these URLs up.
date: 2018-03-22 1:00:00
tags:
- docfx
- netlify
url: /blog/pretty-docfx-urls-netlify
---

Since [I am now jobless](/blog/new-adventures), one of the many things I am doing to keep busy is to help [Kévin Chalet](http://kevinchalet.com/) out with the documentation for his [OpenIddict project](https://github.com/openiddict) - a free, open-source OpenID Connect server for ASP.NET Core.

We are using [DocFX](http://dotnet.github.io/docfx) to generate the documentation, but one of the issues with DocFX is that the generated output contains the `.html` extension in the URL for all pages. 

For example:

![](/images/blog/2018-03-22-pretty-docfx-urls-netlify/ugly-url.png)

This does not quite sit right with Kevin (and neither with me), so I looked into it. I am aware that Netlify has a **Pretty URLs** feature as part of its **Asset Optimization** feature set.

I looked into this before for the OpenIddict documentation, but at that time I did not get it to work. On that occasion, however, I created the website by copying the DocFX generated files directly from my computer to Netlify.

I looked into it again, but this time around, I configured Netlify to deploy from GitHub and checked the DocFX generated documentation into a GH repo. I then ensured that I configure Netlify to make the URLs pretty:

![](/images/blog/2018-03-22-pretty-docfx-urls-netlify/netlify-configuration.png)

With everything in place, I triggered a build for the website, and sure enough, Netlify did its thing. No more ugly URLs!

![](/images/blog/2018-03-22-pretty-docfx-urls-netlify/pretty-url.png)

I am not quite sure how this works technically, but I suspect there is some sort of URL rewriting going on. However, it is more than just that. Netlify also fixes all internal links in the website to redirect to the pretty URL.

For example, notice the URL for the link when I hover over the **Migration Guide** link. Without Netlify, you will notice that the URL for that link contains the `.html` extension:

![](/images/blog/2018-03-22-pretty-docfx-urls-netlify/ugly-url-2.png)

Here is the Netlify version. Notice that the URL for the link is the pretty one, without the `.html` extension:

![](/images/blog/2018-03-22-pretty-docfx-urls-netlify/pretty-url-2.png)

Just one more reason [I like Netlify](/blog/why-i-like-netlify) 😎
