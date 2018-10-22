---
title: "Using ANSI colour codes in .NET Core Console applications"
description:
  Demonstrates how to use ANSI colour codes in .NET Core console applications, as well as the limitations of this on Windows.
date: 2018-04-05
tags:
- dotnet
- dotnet core
- cli
url: /blog/using-ansi-color-codes-in-net-console-apps
---

## Introduction

ANSI escape codes allow you to perform a range of manipulations in console applications. They allow you to do things such as move the cursor around, displaying text in different colours, display text in bold, underline, etc, as well as various other things.

In this blog post, I will look specifically at the colour codes and how you can use them in .NET Core console applications. I will also discuss why you should probably avoid using them - especially when developing applications for Windows.

**Update on 22 October 2018**: Manuel Riezebosch notified me of a library he wrote called [Crayon](https://github.com/riezebosch/Crayon) that wraps the techniques I describe in this blog posts. Please check it out if you want to use these techniques.

## A brief overview of using the ANSI codes

ANSI escape codes are character sequences which you can print to the terminal (or console) window to give the terminal certain instructions. As discussed in the introduction, it allows you to perform various actions in the terminal, but for now, we'll focus on the use case for printing text in specific colours.

To tell the terminal to switch to using a specific colour we need to send the escape sequence for that colour. Let's say, for example, that we want to switch the terminal foreground colour to red, we can write the escape sequence `ESC[31m` to the terminal.

Escape sequences for colours always start off with `ESC[`, followed by the number for the colour, in this case, `31`, which is red. Finally, we need to close off the sequence with `m`. 

Any text following this will now be printed in red. Finally, you will probably need to reset the colour and for that, you will write the sequence `ESC[0m` to the terminal.

I strongly recommend that you also read [Build your own Command Line with ANSI escape codes](http://www.lihaoyi.com/post/BuildyourownCommandLinewithANSIescapecodes.html). It contains a lot more in-depth information on using ANSI codes.

## Using it in a .NET Core console app

Let's put this in practice with an elementary .NET application. The sample code below writes the escape sequence for red, followed by the text **Hello World**, and finally the escape sequence to reset the terminal colours.

```csharp
static void Main(string[] args)
{
    Console.WriteLine("\u001b[31mHello World!\u001b[0m");
}
```

Note that, the `ESC` Unicode character is `001b` ([see here](https://www.compart.com/en/unicode/U+001B)), and in C# we specify Unicode characters as `\uxxxx` where `xxxx` is character code. Therefore, to write the escape sequence `ESC[31m` in C#, we need to write `\u001b[31m` to the console.

Running the application in [Cmder](http://cmder.net/), I get the expected output:

![](/images/blog/2018-04-05-using-ansi-color-codes-in-net-console-apps/ansi-codes-powershell.png)

But running that same application in the normal Windows Command Prompt gives a whole different output:

![](/images/blog/2018-04-05-using-ansi-color-codes-in-net-console-apps/ansi-codes-command-prompt.png)

It turns out that ANSI escape sequences are [not properly supported on Windows](https://en.wikipedia.org/wiki/ANSI_escape_code#Platform_support). While they will work fine on other platforms such as macOS and Linux, using them in the Windows is problematic. 

In the first screenshot, I used Cmder, which uses ConEmu, which in turn interprets these escape sequences correctly.

## Windows 10 support

Further investigation revealed that Microsoft did, in fact, start adding support for ANSI escape sequences in [Windows 10 version 1511 (Threshold 2)](http://www.nivot.org/blog/post/2016/02/04/Windows-10-TH2-(v1511)-Console-Host-Enhancements). It is, however, not enabled by default and you will need to [explicitly enable it in your application](https://docs.microsoft.com/en-us/windows/console/console-virtual-terminal-sequences).

I found a [Gist containing some sample C# code to do this](https://gist.github.com/tomzorz/6142d69852f831fb5393654c90a1f22e), and updated my code accordingly:

```csharp
class Program
{
    private const int STD_OUTPUT_HANDLE = -11;
    private const uint ENABLE_VIRTUAL_TERMINAL_PROCESSING = 0x0004;
    private const uint DISABLE_NEWLINE_AUTO_RETURN = 0x0008;

    [DllImport("kernel32.dll")]
    private static extern bool GetConsoleMode(IntPtr hConsoleHandle, out uint lpMode);

    [DllImport("kernel32.dll")]
    private static extern bool SetConsoleMode(IntPtr hConsoleHandle, uint dwMode);

    [DllImport("kernel32.dll", SetLastError = true)]
    private static extern IntPtr GetStdHandle(int nStdHandle);

    [DllImport("kernel32.dll")]
    public static extern uint GetLastError();

    static void Main(string[] args)
    {
        var iStdOut = GetStdHandle(STD_OUTPUT_HANDLE);
        if (!GetConsoleMode(iStdOut, out uint outConsoleMode))
        {
            Console.WriteLine("failed to get output console mode");
            Console.ReadKey();
            return;
        }

        outConsoleMode |= ENABLE_VIRTUAL_TERMINAL_PROCESSING | DISABLE_NEWLINE_AUTO_RETURN;
        if (!SetConsoleMode(iStdOut, outConsoleMode))
        {
            Console.WriteLine($"failed to set output console mode, error code: {GetLastError()}");
            Console.ReadKey();
            return;
        }           

        Console.WriteLine("\u001b[31mHello World!\u001b[0m");
    }
}
``` 

This time around, running the application in the Windows 10 Command Prompt gave the desired results:

![](/images/blog/2018-04-05-using-ansi-color-codes-in-net-console-apps/ansi-codes-command-prompt-2.png)

## Should you use it?

So this leaves the question: Should you use it?

The answer for that is **probably not...**

Unless you have exact control over the environment in which your console application will be running, it probably is not worth taking the risk that it may not work on some users' computers. So, for now, if you want to use nice colours in your .NET Core console applications, it is probably best to stick to using `Console.ForegroundColor` and `Console.BackgroundColor` to manipulate colours in your console applications.

## But what about Node.js?

During my investigations into using ANSI codes in C# console applications, I stumbled across Node.js libraries such as [Chalk](https://github.com/chalk/chalk), which appear to work fine on Windows:

![](/images/blog/2018-04-05-using-ansi-color-codes-in-net-console-apps/colors-node-windows.png)

Looking through their source code I also could not find any specific accommodation for Windows, so I was initially a bit stunned as to why this works on Windows, but doing the same thing from my C# applications does not work.

It turns out that Node.js actually [provides a proxy layer in Windows](https://github.com/chalk/chalk/issues/98) which will emulate the ANSI escape sequences correctly when Node.js applications run on Windows.

It would be cool if Microsoft could do the same sort of thing, to enable us .NET developers to more confidently use the ANSI escape sequences on Windows, wouldn't it?