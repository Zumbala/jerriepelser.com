---
date: 2017-02-18T00:00:00Z
description: |
  The TinyPNG CLI is a command-line tool that allows you to shrink images using the TinyPNG API.
tags:
- tinypng
- blogging
title: Compress Images using the TinyPNG CLI
url: /blog/compress-images-using-tinypng-cli/
---

I am busy doing a few SEO related optimizations on my blog and one of the actions I am taking is to compress (or shrink) all the images for my blog. I came across the TinyPNG CLI tool which allows you to easily compress all images from the Command Line.

This is a quick introduction to the tool.

## Background

I am systematically going through my blog trying to identity SEO related issues and fix those. One of the areas I identified is that I never bothered to optimize the images (typically screenshots) which I use in my blog.

The total size of all the images on my blog was about 24MB. 

The first tool I tried was a Windows tool called [PNGGauntlet](https://pnggauntlet.com/). It is a free Windows Utility, so I downloaded it and set it to work. The total time it took to run through all the images was around 1 hour and it brought the over all size down by about 8MB, so down to around 16MB for all the images.

At the same time I tried another utility on my Mac called [ImageOptim](https://imageoptim.com/mac). This one fared a little bit better, and after about an hour of work it reduced the overall size of all the images by about 9MB - so down to around 15MB.

## TinyPNG

At [Auth0](https://auth0.com/) we always use TinyPNG to compress images we create for the documentation and tutorials. I tried that quickly on a few images and got much better compression ratios that either PNGGauntlet or ImageOptim.

The only problem was that the web interface allows you to do a maximum of 20 images at a time. It is also cumbersome because it means I have to upload each image individually, then download it after it has been compressed and finally copy it over the old image.

I noticed however that they have a [Developer API](https://tinypng.com/developers) available which gives you 500 free images a month.

Only I was not really in the mood for writing my own app that works with the developer API.

## tinypng-cli

So I did a quick search for a TinyPNG CLI and came across [tinypng-cli](https://www.npmjs.com/package/tinypng-cli). It is a Node.js package, so make sure you have Node.js installed and then install the tinypng-cli package globally:

```text
npm install -g tinypng-cli
```

Next, [sign up for a Developer API Key](https://tinypng.com/developers) which they will email to you in a few seconds.

Once you have the API key you can run the command line utility. Tell it to look in the current folder (`.`), and also all sub-folders (`-r`) and pass along the API Key you received in the `-r` option, e.g.

```text
tinypng . -r -k YOUR_API_KEY
```

Compressing all 450 images took under a minute (as opposed to an hour with the desktop applications!), and the total size of the images came down from 24MB to 8.5MB. That is a saving of almost 16MB - more than double what either of the desktop applications achieved.

With the command line utility installed it is now much easier in future to quickly compress all images I do for the blog.