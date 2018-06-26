---
date: 2017-08-06T00:00:00Z
description: |
  VS Code is a great tool for developing ASP.NET Core applications. Here's looking at some resources which you will find useful.
tags:
- .net core
- csharp
- vs code
title: Using Visual Studio Code for C# (.NET Core) development
url: /blog/using-vscode-for-csharp-development/
---

Lately I have been using my Mac as my main daily driver. I still spin up Parallels from time to time when I need to do something inside Visual Studio, but for most of my C# work I use VS Code (and .NET Core).  This blog post contain some notes about using VS Code for your C# and .NET Core development which may make your life easier.

> As a side note, I have also actually started looking seriously into [JetBrains Rider](https://www.jetbrains.com/rider/) as I miss the powerful refactoring capabilities from Resharper.

## Installation

Installation of VS Code and the .NET Core command line tools are pretty straight forward and many resources are available. For installing VS Code you can refer to the [Setup section](https://code.visualstudio.com/docs/setup/setup-overview) of the official documentation. It provides step-by-step instructions for Windows, Linux and Mac.

For dowbloading the .NET Core SDK you can refer to the [Download section](https://www.microsoft.com/net/download) of the .NET website. You can also refer to the [Get Started with .NET Core](https://www.microsoft.com/net/core) document which contains more detailed steps. It also contains a nice video walkthrough of installing both VS Code and .NET Core.

### Documentation

The Visual Studio website contains pretty amazing [documentation](https://code.visualstudio.com/docs) which will help you get started with the basics of using the editor. Also refer to [Working with C#](https://code.visualstudio.com/docs/languages/csharp) to get a better understanding of the C# support inside VS Code.

## Extensions

Out of the box, VS Code has very limited support for C#. It does however allow you to enrich the standard functionality by installing extensions. You can discover extensions either from inside VS Code, or from the [Visual Studio Code Marketplace](https://marketplace.visualstudio.com/VSCode).

The [VS Code Extension Marketplace](https://code.visualstudio.com/docs/editor/extension-gallery) document contains more information about how to install extensions. Also refer to [Using Extensions in VS Code](https://code.visualstudio.com/docs/introvideos/extend) which contains a video walktrough of installing extensions.

Below are some extensions I recommend you install to get the most from VS Code for C# (and ASP.NET Core) development.

### Extensions for C#

* [C#](https://marketplace.visualstudio.com/items?itemName=ms-vscode.csharp). This extension adds Syntax Highlighting, Intellisense, Go To Definition, Find All References, etc. It also add debugging support for C# applications.

    > For Debugging I also suggest you refer to [Instructions for setting up the .NET Core debugger](https://github.com/OmniSharp/omnisharp-vscode/blob/master/debugger.md)

* [C# Extensions](https://marketplace.visualstudio.com/items?itemName=jchannon.csharpextensions). This adds functionality to easily add a new C# class or Interface. This is useful because just creating a new file inside VS Code simply creates an empty file. This extension creates a new file for a `class` or `interface` for you, and also sets up the correct `namespace` for your project, as well as include the basic `import` statements.

* [C# Snippets](https://marketplace.visualstudio.com/items?itemName=jorgeserrano.vscode-csharp-snippets). Contains many useful C# code snippets.

* [C# XML Documentation Comments](https://marketplace.visualstudio.com/items?itemName=k--kato.docomment). Makes adding XML comments to your code a much more pleasant experience.

### Extensions for ASP.NET Core

* [ASP.NET Core Snippets](https://marketplace.visualstudio.com/items?itemName=rahulsahay.Csharp-ASPNETCore). Contains snippets for easily constructing ASP.NET Core Controllers and Actions.

* [WilderMinds' ASP.NET Core Snippets](https://marketplace.visualstudio.com/items?itemName=wilderminds.wilderminds-aspnetcore-snippets). An alternative to the *ASP.NET Core Snippets* extension adding a very similar set of snippets.

* [ASP.NET Helper](https://marketplace.visualstudio.com/items?itemName=schneiderpat.aspnet-helper). Adds Intellisense inside Razor view pages.

* [HTML Snippets](https://marketplace.visualstudio.com/items?itemName=abusaidm.html-snippets). Adds support for HTML syntax highlighting, as well as HTML snippets.

    > BTW, VS Code also includes amazing [built-in Emmet support](https://code.visualstudio.com/docs/languages/html#_emmet-snippets) inside VS Code, which makes coding of HTML much faster.

* [IntelliSense for CSS class names](https://marketplace.visualstudio.com/items?itemName=Zignd.html-css-class-completion). Provides CSS class name completion for the HTML class attribute based on the CSS class definitions in your project.

### Others

* [Git Extension Pack](https://marketplace.visualstudio.com/items?itemName=donjayamanne.git-extension-pack). If you are using Git, then this is an [Extension Pack](https://code.visualstudio.com/blogs/2017/03/07/extension-pack-roundup) which installs 5 different extensions which adds a lot of very useful Git capabilities to VS Code (over and above what is already [built in](https://code.visualstudio.com/docs/editor/versioncontrol)).

* [Darcula Theme](https://marketplace.visualstudio.com/items?itemName=rokoroku.vscode-theme-darcula). My favourite VS Code [theme](https://code.visualstudio.com/docs/getstarted/themes).

## Refactoring

One major pain point coming from Visual Studio (and Resharper) is the lack of proper code refactoring tools. There is however some hope, as you can - with some manual effort - get some of the Visual Studio Roslyn-based refactoring extensions to work inside VS Code. 

For more information on how to do this, please refer to [Using Roslyn refactorings with OmniSharp and Visual Studio Code](https://www.strathweb.com/2017/05/using-roslyn-refactorings-with-omnisharp-and-visual-studio-code/) by Filip W. It is not a perfect solution, but it is a good start.
