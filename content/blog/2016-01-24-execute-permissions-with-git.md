---
date: 2016-01-24T00:00:00Z
description: |
  Ran into a "Permission Denied" problem recently on Linux with executing a script from Jenkins, and this is how it was resolved.
tags:
- git
- linux
- jenkins
title: Assign execute permissions with Git
url: /blog/execute-permissions-with-git/
---

Recently I had to configure a build on Jenkins, and ran into an issue with a shell script that did not want to execute and failed with a "Permission Denied" error.

Being new to the Linux world I reached out to a colleague and it turned out the solution was an easy one. It is new for me, so I am sharing it so it can maybe help you in the future.

The solution is to use the Git [update-index](https://git-scm.com/docs/git-update-index) command to assign the execute permissions.

Let's say the bash script in question is named `foo.sh`, then go to your shell (or Git shell if you're on Windows like me) and execute the following command:

``` text
git update-index --chmod=+x foo.sh
```

This will assign execute permissions to the bash file. After that you can commit the changes to the repo.

Hope this helps someone out there :)