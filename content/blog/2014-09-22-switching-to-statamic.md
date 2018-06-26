---
date: 2014-09-22T00:00:00Z
description: |
  Chronicles my experiece in moving my blog from Hexo on Azure to Statamic running on Linux.
tags:
- statamic
title: Switching to Statamic
url: /blog/switching-to-statamic/
---

I have been running beabigrockstar.com on a static website engine called [Hexo](http://hexo.io/) for quite a while after I finally got fed up with WordPress which contained just too much bloat. Hexo served me well but one of the issues I faced was that I could no longer schedule posts, which was a feature of WordPress I used quite often. When I write blog posts I like to sit down and get a bunch of them done at the same time, but I also like to spread the publishing of those blog posts out over a week or two.

I considered moving to [Ghost](ghost.org) once they make post scheduling available but in the meantime I came across a product called [Statamic](). It has many of the same advantages of normal static website generators in that I can write in Markdown and also put my entire website under source control, but it also had a dynamic element to in because it was written in PHP. Because of the dynamic capabilities it also has the feature to write and schedule future blog posts. There is no free version available so I bought the Personal edition for $29, and I have not been disappointed so far. 

> As a side note, I have actually developed a bit of an aversion to a lot of free software over the past year because either support is lacking, or they are tracking and data mining my every click, email, like and dislike to build up a profile about me so they can advertise more crap I do not need to me (here's looking at you Google and Facebook...). I am still using Google Analytics (I have a lot of historical info on there) but for the rest I am now using service like FastMail, Office 365 etc. And I guess you can now add Statamic to that list. 

Getting my content across from Hexo was quite easy because both products use Markdown and both rely on [YAML Frontmatter](http://statamic.com/learn/core-concepts/content-files) for meta information about the content. The directory structures are a little bit different for both which means my images for the blog posts are now hosted in a different folder but that was easily fixed with a find and replace. The location of some of the posts also changed so I extracted all the posts from the sitemap file and created the appropriate rewrite rules in my web.config (at the time this website was hosted on Azure, but a bit more about that further down this post).

For the blog theme none of the Statamic themes available tickled my fancy, so after a fairly quick search on [Theme Forest](http://themeforest.net/) I came across a nice HTML theme I liked called [Readable](http://themeforest.net/item/readable-blog-template-focused-on-readability/7499064). Looking through the standard Statamic themes and their online documentation (which is very good) I was able to create a proper Statamic theme from the Readable HTML theme in a couple of hours. I also applied some [VS Web Essentials](http://vswebessentials.com/) magic to the theme to combine and minify my SASS and Javascript files. For testing the website locally on my machine I used [MAMP](http://www.mamp.info/).

The previous static version on this site was hosted on Azure and knowing that Azure supported PHP I created another test website on Azure to host the Statamic version. I set up Git deployment and was able to test that the website was working correctly and all the HTTP redirects was also functioning correctly so changed over my DNS entries to the new website. 

## The horror of PHP on Azure
Some time later I decided to take a peek at my [Pingdom](http://www.pingdom.com) account and was greeted by a nastly little graph.

![](/assets/images/switching-to-statamic/pingdom-horror.png)

Seeing as I went from a purely static website to one which was running on PHP I was expecting to take a bit of a knock in response times but this was crazy. Response times went from around 300ms to 2300ms. I had a more detailed look with [Pingdom's Website Speed Test tool](http://tools.pingdom.com/) to see exactly where the problem was.

![](/assets/images/switching-to-statamic/pingdom-horror-2.png) 

See that long yellow part which I highlighted in red? That is waiting for the server. For some reason Azure took a very long running those PHP pages. I bumped my Azure website up from Small (1 core, 1.75BG memory) to Medium (2 cores, 3.5GB memory) but did not see any significant improvement.

## Going the Linux route
I stayed away from Linux with my website because I had the Azure credits available to host the website and setting up deployment from Github to Azure was super quick and easy. Also Azure have always performed plenty good enough for my requirements. But this was no good, and I did not have the time to try and understand why all this was happening, so I decided to try and move the website to Linux. My friend Jirapong runs a software development company here in Chiang Mai called [Banana Coding](http://www.bananacoding.com/) and he offered to help me set it up on Linux.

I initially tried going the [Digital Ocean](https://www.digitalocean.com/) route but for some reason had issues connecting to the Linux box with the Putty SSH client (the connection was slow and dropped the whole time). Jirapong experienced the same issues from OSX so in the end we went with [Linode](https://www.linode.com/). Setting up a machine was a bit more cumbersome than with Digital Ocean but I had Jirapong's Linux skills on hand to do it for me. The documentation on the Statamic website is also good in explaining what needs to be done.

The website is now running on a 1GB Linux box at Linode. We set up automated deployment with [dploy.io](http://dploy.io/) which runs a shell script on the server to get the latest version of the website whenever I do a commit to Github. The whole process was obviously a bit more cumbersome than with Azure, but the results speak for themselves:

![](/assets/images/switching-to-statamic/pingdom-happy.png)

The response times in Pingdom is back to around 550ms. Still room for improvement as can be seen below:

![](/assets/images/switching-to-statamic/pingdom-happy-2.png)

Those CSS and Javascript files are way to big for my liking, but that is because the Readable template is built on top of Bootstrap. The majority of the Bootstrap components are however not used so I just need to go through the SASS files and Javascript bundles and exclude all the stuff which adds bloat and no value.

A few other things I need to note as far as the switch to Linux is concerned:
* The filenames are case sensitive so I had a few problems where I linked to images inside my posts with a lowercase .png extension while the file was saved with an uppercase .PNG extension (which the Windows Snipping tool does automatically).
* I went through a bit of effort in setting up the rewrite rules in my web.config but in switching over to Linux and Apache I had to change over to a .htaccess file. There is a website called [tohtaccess.com](http://tohtaccess.com/) which helps you in converting from a web.config to a .htaccess file and it worked great. BTW, [there is also a tool which does the conversion the other way around](http://htaccesstowebconfig.com/).

## Conclusion
I am very happy with Statamic so far. The product is easy to understand and the documentation is great. The one experience I have had with their support has also been good. The workflow of writing blog posts in Markdown, pushing it up to Github and having it deployed automatically suits me as a developer perfectly. It is the way I like to do things. 

As for the bad performance on Azure, I do not know what the exact reason for that was. Maybe there was something I was doing wrong, or maybe Azure is just horrible at serving PHP pages. At least I got to learn some new stuff in moving over to a Linux box and learning something new is always a good thing.