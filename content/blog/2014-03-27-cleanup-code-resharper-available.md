---
date: 2014-03-27T00:00:00Z
description: |
  Sometimes you may find that the Resharper Code Cleanup option is not available. This discusses the likely cause and how to work around it.
tags:
- nuget
- resharper
title: Cleanup Code in Resharper not available
url: /blog/cleanup-code-resharper-available/
---

While writing my previous blog post about using Google Authenticator with ASP.NET Identity, I ran into a strange issue with [Resharper](http://www.jetbrains.com/resharper/) where the "Code Cleanup" option would not be available for certain files.

![](/assets/images/2014/03/Screen-Shot-2014-03-24-at-4.55.59-PM.png)

I tried the obvious things such as resetting all the Resharper options and even uninstalling and reinstalling Resharper, but neither made any difference so I got in contact with Jetbrains support to try and figure it out.

During the discussion I noticed that the option seemed to be unavailable on files which were already under source control, but files which I added to the project and was not under source control yet did not seem to have the problem.  So I committed the newly added files to test out that theory but it made no difference - the Cleanup Code option was still available for those particular files, but it was unavailable for the majority of the other files in the project.

I cloned the repository to a completely separate folder to make sure there wasn't something else strange going on, but noticed the exact same behaviour.  The Cleanup Code option was available for a handful of files in the project, but not for the majority of the files.

After a little bit more going back and forth with Jetbrains support they finally figured out the cause of the problem and the solution.  It turns out that for that particular project I installed the [ASP.NET Identity Samples](http://www.nuget.org/packages/microsoft.aspnet.identity.samples) package from Nuget.  That Nuget package replaced basically all of the files in my project (which was a basic ASP.NET MVC application) with new files (as can be seen in the screenshot below from the package's content folder)

![](/assets/images/2014/03/Capture.png)

Resharper understands which files has been added to the project via a Nuget packages and treats those files as non-user code.  Treating these files as non-user code basically excludes the files from certain code analysis and code inspection operations.  This is discussed in more details in the Resharper [documentation regarding generated code](http://www.jetbrains.com/resharper/webhelp/Reference__Options__Code_Inspection__Generated_Code.html).

Generated code is automatically excluded from these sort of inspections as the developer have very little control over what gets generated and you do not want to see a whole lot of errors and warnings from the Resharper code inspection tools for source code which is outside of your control.  Even if you fix the errors or warnings in generated code the chances are good that whatever tool generated the code will overwrite it again with non-complaint code somewhere in the future.

Well, the same goes for source code files and other content files which are added to your project via Nuget packages.  You can fix the code inspection warnings, but as soon as you upgrade the package those fixes are probably going to be overwritten in any case.  Resharper therefore excludes those files from these inspections.

The only problem is that somehow the Cleanup Code option is also not available for those files.  I guess the reasoning from Jetbrains is that there is no use cleaning up the code in these files as the content are likely to be overwritten again somewhere in the future.

For now the only fix is to fool Resharper into believing that those files were not added to the project by the Nuget package by editing your `packages.config` file manually and either commenting out or removing the reference to the particular Nuget package.  This is obviously also not ideal and there is an [issue logged in the Jetbrains issue tracker](http://youtrack.jetbrains.com/issue/RSRP-371787) regarding this.  It seems they may address this in a more elegant way in the next major release.

In my case I went ahead and commented the reference to the Nuget package out as you will probably not ever want to upgrade that particular package in any case because it will replace almost all the files in your project with updated ones and overwrite all the changes you have made in the meantime.