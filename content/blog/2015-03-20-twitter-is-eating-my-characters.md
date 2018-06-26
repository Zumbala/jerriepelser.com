---
date: 2015-03-20T00:00:00Z
description: |
  When using the word ASP.NET in Twitter, Twitter will use 22 characters instead of 7. This blog post shows you how to fool Twitter.
tags:
- aspnet
- twitter
title: 'Hey ASP.NET developers: Twitter is stealing from your 140 characters'
url: /blog/twitter-is-eating-my-characters/
---

## Introduction 
A picture is worth a thousand words, so check this out:

![](/assets/images/2015-03-20-twitter-is-eating-my-characters/problem-tweet.png)

I have a tweet with the words "ASP.NET", which is 7 characters and yet according to Twitter I have used up 22 characters already...

Twitter uses a built in link shortening service which shortens all links using its own link service (http://t.co). It does this for a number of reasons which they describe in the document [About Twitter's link service](https://support.twitter.com/articles/109623). The important 2 things to understand is the following:

1. Any link you post to Twitter will take up 22 characters. It does not matter whether that link is shorter or longer that 22 characters.
2. There is no way to opt out of it

Now the bad news for ASP.NET developers is that everytime you post a tweet with the text "ASP.NET" inside of it, Twitter will recognise that as a URL and convert it using its own link shortening service. So instead of using up 7 characters, it uses 22 characters, meaning everytime you tweet something about ASP.NET you lose 15 characters for no good reason at all. Heaven forbid you use ASP.NET twice in a tweet, because just like that you will be out of 30 characters!

I have been aware of this for a long time, but since I have started with my [ASP.NET Weekly](http://www.aspnetweekly.com/) newsletter this has started to bother me all the more. See, along with the ASP.NET newsletter which goes out once a week, I also send out a bunch of tweets on the [ASP.NET Weekly Twitter account](https://twitter.com/aspnetweekly). So you can imagine those tweets contain the words ASP.NET quite a lot which means Twitter is stealing an excessive amount of characters from me on a weekly basis.

On the album Rattle and Hum, U2's Bono introduced the song Helter Skelter with the words:

> "This is a song Charles Manson stole from The Beatles. We're stealing it back."

Well, ASP.NET developers, it is time to declare:

> "These are 15 characters Twitter stole from me. I am stealing them back."

## Stealing them back

It turns out that I have actually already solved this problem for the newsletter itself. Many email clients, like Outlook for iOS, will also pick up on the words ASP.NET in the newsletter and turn it into hyperlinks. This is something some of the newsletter readers did not like, and they are right. It looks ugly.

So I did a bit of Googling and the way I work around it for now is to insert the character `&#8203;` after the period in the word ASP.NET. So instead of using `ASP.NET` in my newsletter, in the HTML source of the newsletter I acually write `ASP.&#8203;NET`. When the HTML renders it looks like this:

ASP.&#8203;NET

The reason this works is because that is the [Unicode Character 'ZERO WIDTH SPACE'](http://www.fileformat.info/info/unicode/char/200b/index.htm), so when it is rendered in HTML it does not display, and does not take up any extra space. See, it looks like I wrote the word "ASP.NET", but when you look at the source code for this page you will see that I inserted the `&#8203;` character after the period.

So if you do not want Twitter to steal any more of your characters, the next time your write a tweet you can just come and copy the text above and paste it in your tweet and Twitter won't think it is a link which it must shorten.

Because of the special character I inserted, the word "ASP.NET" is now obviously 8 characters long instead of 7, but I'll take that any day over 22 characters :)

![](/assets/images/2015-03-20-twitter-is-eating-my-characters/asp-net-trick.png)

Of course you can also have a bit more fun with it. Go to the [Symbols for Twitter webpage](http://twsym.com/), and highlight and copy one of those symbols before the period in ASP.NET. 

How about we use the heart symbol, which obviously shows our love for ASP.NET:

![](/assets/images/2015-03-20-twitter-is-eating-my-characters/heart-asp-net.png)

Which renders out pretty damn nicely in a tweet :)

![](/assets/images/2015-03-20-twitter-is-eating-my-characters/heart-asp-net-rendered.png)

So let's use this opportunity to start spreading the love for ASP.NET