---
date: 2014-07-28T00:00:00Z
description: |
  How to determine which Visual Studio command is assigned to a specific keyboard shortcut.
tags:
- visual studio
title: Determine which command is assigned to a keyboard shortcut in Visual Studio
url: /blog/determine-which-command-is-assigned-to-a-keyboard-shortcut-in-visual-studio/
---

I recently was browsing through the list of [default keyboard shortcuts](http://msdn.microsoft.com/en-us/library/da5kh0wa.aspx) for Visual Studio 2013 and came across a combination of keys which allows you to easily zoom in and out in the code editor. These are `Ctrl+Shift+.` to zoom in and `Ctrl+Shift+,` to zoom out.  

I tried them out quickly and noticed that only the combination for zooming in was working. I first thought that it was an issue with the layout of the International keyboard on the Macbook Pro (it has a few quirks), but then realised that it was probably Resharper which has taken over that specific keyboad combination.

I headed over to the Keyboard options in Visual Studio to try and figure out what was going on but was a bit stumped as there seemed no obvious way to figure out which command an existing keyboard combination was assigned to (if any). It seemed that the only was was to browse through all of the commands and there was not way I was going to try that.

I then noticed a greyed out area at the bottom of the Keyboard options section which stated "Shortcut currently used by":

![](/assets/images/vs-keyboard-shortcuts/keyboard-options-1.png)

So I selected the first available command and pressed the keyboard combination `Ctrl+Shift+,` and there it showed me the exact command which is now assigned to that keyboard combination.

![](/assets/images/vs-keyboard-shortcuts/keyboard-options-2.png)

Mystery solved. 

So remember this little trick next time a keyboard shortcut is not behaving the way you think it should.