---
date: 2015-09-08T00:00:00Z
description: |
  A call for contributors to help with the new OAuth providers for ASP.NET 5
tags:
- asp.net
- oauth
title: Calling contributors for OAuth providers for ASP.NET 5
url: /blog/calling-contributors-for-oauth-providers/
---

## Introduction

Almost two years ago I did a fun little experiment to see whether I could develop a [Google+ OAuth provider for ASP.NET MVC 5](http://www.jerriepelser.com/blog/google-oauth-sign-asp-net-identity). I then followed it up and created [Yahoo and LinkedIn providers as well](http://www.jerriepelser.com/blog/introducing-the-yahoo-linkedin-oauth-security-providers-for-owin) and decided that it was probably time to turn it into an open source project.

The [resulting project](https://github.com/RockstarLabs/OwinOAuthProviders) has been fairly successful and has 38 providers with 284 commits from 38 different contributors. The total number of downloads for [the Nuget package](https://www.nuget.org/packages/owin.security.providers) is at this moment just shy of 40,000 downloads.

I am however not too good at reviewing pull requests and I have let various little things slip through, and I have also not enforced any kind of consistent coding standards across the board. I guess initially I was just too grateful that people wanted to contribute to my open source initiative and I did not want to start nitpicking and put them off. In hindsight however, this was wrong.  

## ASP.NET 5

Well, with ASP.NET 5 it is time for a new beginning and there is a new Sheriff in town. His name is [Kévin Chalet](https://twitter.com/PinpointTownes), a.k.a. [PinpointTownes](https://github.com/PinpointTownes).

Kévin is the man behind the new [generic OAuth provider](https://github.com/aspnet/Security/tree/dev/src/Microsoft.AspNet.Authentication.OAuth) in ASP.NET 5 about which I [have blogged before](http://www.jerriepelser.com/blog/advanced-configuration-in-the-aspnet5-generic-oauth-provider). 

Kévin has also started an initiative to develop a set of OAuth providers specific to the various services - similar to what I did for ASP.NET MVC 5. It is available on Github at [aspnet-contrib/AspNet.Security.OAuth.Providers](https://github.com/aspnet-contrib/AspNet.Security.OAuth.Providers).

First perhaps, let us answer an fairly obvious question: **Why specific implementations when there is a generic OAuth implementation in ASP.NET 5?**

The answer is because the generic implementation only takes you so far. Sure, it does the basic authentication and you can get an Access token. For it to integrate properly with ASP.NET Identity however you need to do a bit of extra work, namely to retrieve information about the current user from the underlying service. I demonstrated this in my follow-up blog post entitled [Advanced configuration in the ASP.NET 5 Generic OAuth Provider](http://www.jerriepelser.com/blog/advanced-configuration-in-the-aspnet5-generic-oauth-provider).

Specifically, ASP.NET Identity relies on the `NameIdentifier` claim to be set. The value for this claim must be a unique identifier for the user on that specific service provider - typically this is a unique integer or GUID value. To retrieve that value involves making an extra API call to the relevant service provider, and the specifics of it differs from one service provider to the next. Each service provider has a different API endpoint and the actual data structure which they return will also be different. 

Now sure, you can use the generic provider and then go and write this code yourself like I did in the [Advanced configuration in the ASP.NET 5 Generic OAuth Provider](http://www.jerriepelser.com/blog/advanced-configuration-in-the-aspnet5-generic-oauth-provider) blog post, but why go through this pain when you can just include a small Nuget package that does all this work for you?

So far Kévin has done the provider for GitHub and I have done the ones for Yahoo, LinkedIn and WordPress. 

What we need are people to help contribute the rest. We can do it between ourselves but that will take time as we both have other commitments as well. If we can get some help from the community it would be awesome. Especially if you have written a provider for my old OWIN OAuth providers project, then I ask you to please consider doing the one for ASP.NET 5.

The code is much simpler now thanks to the base classes doing most of the work. You can look at the implementation of the [GitHub provider](https://github.com/aspnet-contrib/AspNet.Security.OAuth.Providers/tree/dev/src/AspNet.Security.OAuth.GitHub) to see what is required. Most of the classes just inherit from the base classes and the only real work you need to do is  in the [AuthenticationHandler implementation](https://github.com/aspnet-contrib/AspNet.Security.OAuth.Providers/blob/dev/src/AspNet.Security.OAuth.GitHub/GitHubAuthenticationHandler.cs) to override the `CreateTicketAsync` method to call the appropriate API to to set the correct claims

The most important claim is the `NameIdentifier` as I stated before, as ASP.NET Identity requires this. This should be the unique ID for the user on that particular service.

Sometime you will need to do a bit of extra work. For example, in the Yahoo provider I [had to also override](https://github.com/aspnet-contrib/AspNet.Security.OAuth.Providers/blob/dev/src/AspNet.Security.OAuth.Yahoo/YahooAuthenticationHandler.cs) the `ExchangeCodeAsync` method because Yahoo does things a bit differently...  

This is a nice opportunity for you to get some open source contributions under the belt. Kévin has worked closely with the ASP.NET 5 team on many of his own contributions, and I am learning a lot from him on how to manage various things on an open source project. I am sure you can learn a lot from him too.

## How to do this

So, how do you do this? 

1. Create a fork of [the repository](https://github.com/aspnet-contrib/AspNet.Security.OAuth.Providers). 
2. Clone your fork and make sure you can complile locally. The [Readme file](https://github.com/aspnet-contrib/AspNet.Security.OAuth.Providers/blob/dev/README.md) should help you. Basically all you need to do is a `dnvm upgrade -u` and `dnu restore`. If you run into other issues, please update the Readme file with instructions.
3. Look at what [providers are available](https://github.com/aspnet-contrib/AspNet.Security.OAuth.Providers/issues) for you to pick up. If you see one you want to work on, leave a comment stating that you are picking it up so we don't get people working on the same things. If you don't see the provider you want to work on listed, then just create an issue for it and then leave a comment that you're implementing it.  
4. Create a branch on your fork for developing the provider and start coding. Use the [GitHub provider](https://github.com/aspnet-contrib/AspNet.Security.OAuth.Providers/tree/dev/src/AspNet.Security.OAuth.GitHub) as a reference. This time around we are doing each provider as its own separate project.
5. Stick the the existing coding conventions, please! I have created a Resharper setting file for the solution, so I you have Resharper it should help you with this. If not - see how the existing code is formatted and stick to it please. Kévin likes to put his opening braces on weird places.... :P
6. Once you have finished please create a pull request.  

I also just want to say that if you are submitting a PR and then Kévin (or myself) leave a long list of things for you to fix which may seem like nitpicking, please don't be discouraged. This is part of the normal process for collaborative development. Project owners sometimes will point out the smallest things, or perhaps even decline a PR, but they do so because they have the best interest of the project in mind.

Also, if you don't agree with the coding standards or naming conventions which the project owner prescribes, please don't get in an argument over it. In my lifetime I have learned that sticking to the coding standards is far more important than what the standards are. If you don't agree with the standards, bear the pain, stick with it and push through it ;)   

If you have any issues, there is a [Gitter channel](https://gitter.im/aspnet-contrib/AspNet.Security.OAuth.Providers) set up for the project so just ask on there.

I am thanking you in anticipation for your assistance with this :)