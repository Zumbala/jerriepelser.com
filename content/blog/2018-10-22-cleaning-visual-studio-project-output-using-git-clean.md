---
title: "Cleaning Visual Studio build output using the git clean command"
description:
  The "Clean Solution" command in Visual Studio does not clear all the build artifacts. Here's how you can use the Git clean command to remove all unwanted build artifacts
tags:
- visual studio
- git
---

Do you know the _Clean Solution_ command in Visual Studio?

![Clean Solution context menu](/images/blog/2018-10-22-cleaning-visual-studio-project-output-using-git-clean/clean-solution-menu.png)

It turns out it is [a pretty useless command](https://stackoverflow.com/questions/1603242/in-visual-studio-what-does-the-clean-command-do).

I am not sure what it does exactly, but on my computer, it does not seem to ever actually delete any build artifacts. The problem is that from time-to-time things go a bit strange, and Visual Studio picks up some old artifacts - especially when working with Git and switching between a lot of branches.

In situations like these, you want to be able to clean all build artifacts to start fresh, and since the _Clean Solution_ command is useless, a colleague of mine advised me to just use the [git clean](https://git-scm.com/docs/git-clean) command.

Specifically, you can run it with the following parameters:

```text
git clean -xdf
```

The `x` option deletes all **untracked files**, the `d` option deletes all **untracked directories**, and the `f` option will **force** the deletion of these files.

**Be careful though**, as this will nuke all files which are not tracked by Git, such as your user-specific settings files (`*.suo`, `*.DotSettings.user`, `.vs` folder etc.). If you're using the old `.csproj` format that makes use of `packages.config` it will also delete your `/packages` directories meaning you will need to restore all NuGet packages as well.

For me, I don't care too much about all these. By the time things get so bad that I feel I want to nuke all build output, I am more than happy to get rid of all those files as well to ensure that I make a fresh start.