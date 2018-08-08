---
title: "You might not need Angular: Introduction"
description:
  Introduction post to a series of blog posts that demonstrate how you can use the jQuery Unobtrusive Ajax library to achieve interactivity on normal, server-rendered ASP.NET Core applications
tags:
- asp.net core
- jquery
- ajax
---

## Introduction

Client-side JavaScript frameworks are all the rage these days. Frameworks like Angular, React and Vue is what all the cool kids are using.

For me personally, I am very comfortable developing websites in ASP.NET Core. The thing I like about those client-side frameworks, however, is the interactivity you can achieve. And the fact that you don't have to do full page-loads when doing many CRUD operations.

Most of the applications I develop do not have a specific need to be full-blow, interactive, rich, client-side applications. For most of the stuff, a good old server-side rendered page will do just fine. But what I want to do is to be able to sprinkle a little bit of interactivity into some of those web pages to make the user experience better.

In this series of blog posts, I want to show you that you may not need Angular after all. Or React. Or Vue.

You can achieve an amazing level of interactivity using good old AJAX.

BTW, I am not trying to argue that server-side frameworks are better than client-side frameworks, or vice versa. As I said before, you need to use what you are most comfortable and productive with. For me, that is ASP.NET Core. So if you're in the same boat as me, then this series may be useful to you. If you prefer Angular, React or Vue, then that is good as well. Keep on using it.

## jQuery Unobtrusive Ajax

The library I am going to use in this series is the [jQuery Unobtrusive Ajax library](https://github.com/aspnet/jquery-ajax-unobtrusive).

_"What!?"_, you might say. _"You're using jQuery? Dude, it's 2018!"_

So here's the thing. I am not a web designer. I buy websites templates so my applications can look decent. Most of those templates are based on Bootstrap, which includes jQuery in any case. So it is not as if I am adding something extra to the mix. jQuery is already there, so I use it.

_So, back to jQuery Unobtrusive Ajax..._

jQuery Unobtrusive Ajax allows you to mix jQuery's Ajax methods into your application in an _unobtrusive_ way. What that means is that you do not have to call the jQuery Ajax methods directly. You decorate certain HTML elements with standard HTML5 `data-*` attributes, and the jQuery Unobtrusive Ajax library will take care of hooking everything up under the hood and ensuring that the jQuery Ajax methods are being called.

## A better tomorrow

As I am writing this on 8 August 2018, I just watched the [latest ASP.NET Core Community standup](https://youtu.be/7Eh_l7jEcCo?t=47m30s) this morning. At the 47:30 mark, Dan Roth talks about [Server-Side Blazor that was shipped in Blazor 0.5.0](https://blogs.msdn.microsoft.com/webdev/2018/07/25/blazor-0-5-0-experimental-release-now-available/). He goes on to talk about how the team plans to bring this functionality into ASP.NET Core 3.0 where it will be called **Razor Components**.

Where Blazor is (planned to be) a client-side framework, server-side Blazor allows you to run your Blazor components server-side, in your ASP.NET Core application. It takes care of doing UI updates and event handling over a SignalR connection.

It is still early days, so only time will tell whether this happens, but this makes me very excited. From my limited understanding (at this moment) of what this does, it seems that this will address many of the scenarios I will outline in the rest of this series.

So how knows? Maybe there is a better tomorrow just around the corner with a lot of these goodies baked right into the ASP.NET Core framework.

## Conclusion

That's it for this introduction to the series. Stay tuned for future blog posts where I will walk through some practical examples, demonstrating how you can use jQuery Unobtrusive Ajax in your ASP.NET Core applications.