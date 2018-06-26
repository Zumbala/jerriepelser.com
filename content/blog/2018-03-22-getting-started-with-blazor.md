---
title: Getting started with Blazor
description: |
  Blazor is an experimental .NET web framework using C#/Razor and HTML that runs in the browser with WebAssembly. Let's see how you can start experimenting with this today.
date: 2018-03-22 2:00:00
tags:
- blazor
- asp.net core
url: /blog/getting-started-with-blazor
---

## Introduction

**Update 23 March 2018:** A few hours after I released this blog post, Microsoft [released the official Blazor Preview](https://blogs.msdn.microsoft.com/webdev/2018/03/22/get-started-building-net-web-apps-in-the-browser-with-blazor/). This means that configuring the extension and package sources to use the Blazor MyGet Gallery as I describe in this blog post, is not necessary. If, however, you want to work with the nightly builds, then you can still use the Blazor MyGet feed.

Blazor is an experimental .NET web framework using C#/Razor and HTML that runs in the browser with WebAssembly. This is still in a very early stage of development, and if it the ASP.NET Core betas are anything to go by, we are in for a bumpy ride.

If you are unfamiliar with Blazor, then I suggest you start off by reading the following blog posts:

* [A new experiment: Browser-based web apps with .NET and Blazor](https://blogs.msdn.microsoft.com/webdev/2018/02/06/blazor-experimental-project/)
* [Blazor: a technical introduction](http://blog.stevensanderson.com/2018/02/06/blazor-intro/)

## Installing Blazor 

I wanted to start playing around with Blazor, so I headed over to the [Blazor GitHub repo](https://github.com/aspnet/blazor) for more information. There does not seem to be any clear indication of how to get it installed and get a new project going. From initial indications, it appears as though you need to clone the repo build everything locally and then install the Visual Studio extension from the compiled sources. 

I did this, and it got things working, but then discovered a much simpler way by just installing everything from the [Blazor MyGet Gallery](https://dotnet.myget.org/gallery/blazor-dev).

Before you even get started, ensure that you have installed the [Visual Studio 2017 15.7 Preview](https://www.visualstudio.com/vs/preview/).

Go to the [Blazor MyGet Gallery](https://dotnet.myget.org/gallery/blazor-dev) and click on the **Connect to feed** button.

![](/images/blog/2018-03-22-getting-started-with-blazor/myget-connect-to-feed.png)

Select the **VSIX** feed and copy the **Vsix feed URL**.

![](/images/blog/2018-03-22-getting-started-with-blazor/vsix-feed.png)

In Visual Studio 2017 Preview, go to **Tools > Extensions and Updates** and click on the **Change your Extensions and Updates settings** link.

![](/images/blog/2018-03-22-getting-started-with-blazor/extensions-and-updates.png)

Under the **Additional Extension Galleries** section, click the **Add** button. Give it a **Name** of _Blazor_, and specify the VSIX feed URL you copied before as the **URL**. Once you are done, you can click **Apply** and then **OK**.

![](/images/blog/2018-03-22-getting-started-with-blazor/additional-extension-galleries.png)

Select the **Tools > Extensions and Updates** menu again. Go to **Online > Blazor** and click the **Download** button next to **ASP.NET Core Blazor Language Services** extension.

![](/images/blog/2018-03-22-getting-started-with-blazor/extensions-and-updates-2.png)

Once the download is done, you will need to close Visual Studio 2017 Preview for the installation to complete. Afterwards, you can open Visual Studio 2017 Preview again.

## Configure the Nuget Gallery

Go back to the [Blazor MyGet Gallery](https://dotnet.myget.org/gallery/blazor-dev) and click on the **Connect to feed** button again. This time, select the **NUGET** feed and copy the **NuGet V3 feed URL** value:

![](/images/blog/2018-03-22-getting-started-with-blazor/nuget-feed.png)

In Visual Studio 2017 Preview, select the **Tools > Options** menu. Go to the **NuGet Package Manager > Package Sources** section, and add a new package source using the **NuGet V3 feed URL** you just copied from MyGet:

![](/images/blog/2018-03-22-getting-started-with-blazor/package-sources.png)

## Create a new project

To try Blazor out, select the **File > New > Project** menu, and create an **ASP.NET Core Web Application**:

![](/images/blog/2018-03-22-getting-started-with-blazor/new-project.png)

Next, select the **Blazor (ASP.NET Core hosted)** template and click **OK**:

![](/images/blog/2018-03-22-getting-started-with-blazor/select-blazor-template.png)

This will create a project similar to the one below:

![](/images/blog/2018-03-22-getting-started-with-blazor/solution-explorer.png)

Build and run the application, and a browser window will open with your brand new Blazor app:

![](/images/blog/2018-03-22-getting-started-with-blazor/blazor-app.png)

Feel free to play around with the app to get a feel for a Blazor application.

## Something more exciting

The application created by the project template is the standard one I am sure many of you have seen before if you have explored any of the Blazor blog posts or demos. However, it came to my attention that there is something a little more substantial hiding the in [https://github.com/aspnet/samples](https://github.com/aspnet/samples) repository.

Clone the repository and then open the solution in the `/samples/aspnetcore/blazor/FlightFinder` folder. Once again you can build and run the application which will open it in the browser:

![](/images/blog/2018-03-22-getting-started-with-blazor/flight-finder.png)

This app is a much better representation of some of the (early) capabilities of Blazor.

## Conclusion

I am very excited to start playing with this, so keep an eye out for more Blazor blog posts coming up. 

If you want to learn more, check out the Issues on the [Blazor repo](https://github.com/aspnet/blazor). Rainer Stropek has also created a website called [Learn Blazor](https://learn-blazor.com/). Finally, the official MS docs for Blazor is in the works and should appear at some point in the ASP.NET Core docs under the [Client-side development](https://docs.microsoft.com/en-us/aspnet/core/client-side/) section.