---
title: "Determine the ConsoleColor from a 24 bit hexadecimal color code"
description:
  The .NET Console is limited to 16 colors. Learn how you can convert a hexadecimal color code to one of the 16 colors available in the console.
date: 2018-04-17
tags:
- dotnet
- dotnet core
- cli
url: /blog/determine-consolecolor-from-hex-color
---

Using colors in console .NET applications can be tricky. Previously I blogged about [not being able to use ANSI color codes reliably in the Windows console](/blog/using-ansi-color-codes-in-net-console-apps). This time I ran into another interesting challenge.

[In my last blog post](/blog/determine-github-repo-from-current-working-directory), I mentioned that I was working on a .NET Core Global tool which will allow developers to work with GitHub Issues from the command line.

## GitHub Issue labels and colors

One of the features of GitHub Issues is that you can assign labels to any issue. Each label is represented by a color, as you can see in screenshot below:

![GitHub Issue Labels](/images/blog/2018-04-17-determine-consolecolor-from-hex-color/github-label-colors.png)

This makes it easier to identify the different labels when you want are scanning the list of issues in a repository:

![List of GitHub Issues](/images/blog/2018-04-17-determine-consolecolor-from-hex-color/github-issues.png)

## Converting a hex color to a console color

In my GitHub Issues CLI, I want to display the list of labels associated with an issue. I also want to display the label in a color similar to the color used on the GitHub website. I say "similar" because on the GitHub website you can choose pretty much any color to represent a label, but in the console I am [limited to a set of 16 colors](https://msdn.microsoft.com/en-us/library/system.consolecolor(v=vs.110).aspx).

After a quick search on the web, I came across [this StackOverflow answer](https://stackoverflow.com/a/29192463/5192029) which demonstrated what I was after. So I wrote a helper method which takes any hexadecimal color and determines the most appropriate `ConsoleColor` to represent that color:

```csharp
public static class ConsoleColorHelper
{        
    public static ConsoleColor FromHex(string hex)
    {
        int argb = Int32.Parse(hex.Replace("#", ""), NumberStyles.HexNumber);
        Color c = Color.FromArgb(argb);
        
        int index = (c.R > 128 | c.G > 128 | c.B > 128) ? 8 : 0; // Bright bit
        index |= (c.R > 64) ? 4 : 0; // Red bit
        index |= (c.G > 64) ? 2 : 0; // Green bit
        index |= (c.B > 64) ? 1 : 0; // Blue bit
        
        return (System.ConsoleColor)index;
    }
}
```

With that in place, I can set the background color for a label the `ConsoleColor` returned from the `ConsoleColorHelper.FromHex` method:

![Set background color for label](/images/blog/2018-04-17-determine-consolecolor-from-hex-color/github-issues-console-background.png)

This still leaves me with a problem though. I set the `ForegroundColor` to white, but as you can see in the screenshot, that obviously does not work well when the background color is a light color. What I need to do is to set the foreground color based on whether the background color is a light or a dark color.

Thankfully, I found [another Stackoverflow answer](https://stackoverflow.com/a/1855903/5192029) which demonstrates how you can determine a "perceptive luminance", and then set the foreground color based on that.

So with that, here is the updated code for my `ConsoleColorHelper.FromHex` method. Note that I also updated the method signature to return the foreground and background color combination as a [tuple](https://docs.microsoft.com/en-us/dotnet/csharp/tuples).

```csharp
public static class ConsoleColorHelper
{        
    public static (ConsoleColor ForegroundColor, ConsoleColor BackgroundCololr) FromHex(string hex)
    {
        int argb = Int32.Parse(hex.Replace("#", ""), NumberStyles.HexNumber);
        Color c = Color.FromArgb(argb);
        
        // Counting the perceptive luminance - human eye favors green color... 
        double a = 1 - ( 0.299 * c.R + 0.587 * c.G + 0.114 * c.B)/255;

        int index = (c.R > 128 | c.G > 128 | c.B > 128) ? 8 : 0; // Bright bit
        index |= (c.R > 64) ? 4 : 0; // Red bit
        index |= (c.G > 64) ? 2 : 0; // Green bit
        index |= (c.B > 64) ? 1 : 0; // Blue bit
        
        ConsoleColor backgroundColor = (System.ConsoleColor)index;
        ConsoleColor foregroundColor = a < 0.5 ? ConsoleColor.Black : ConsoleColor.White;
        
        return (foregroundColor, backgroundColor);
    }
}
```

As you can see, that fixes the readability of the labels:

![Set foreground and background color for label](/images/blog/2018-04-17-determine-consolecolor-from-hex-color/github-issues-console-foreground.png)

## Source Code

I have no specific source code for this blog post, but the source code for my GitHub Issues CLI can be found at [https://github.com/jerriep/github-issues-cli](https://github.com/jerriep/github-issues-cli).

