---
date: 2017-09-07T00:00:00Z
description: |
  A useful tip for exluding files from just your own local Git repository by using the .git/info/exclude file
tags:
- git
title: How to exclude files from Git only on your computer
url: /blog/exlude-git-files-locally/
---

At [Auth0](https://auth0.com/) we develop a lot of sample applications to show developers how to use our various features inside their applications. These applications invariably require some sort of configuration which is read from a file.

For our Angular Quickstarts for example, there is a file called `auth0-variables.ts.example` which contains an example of the settings required for the application to run.

![](/assets/images/2017-09-07-exlude-git-files-locally/angular-sample.png)

When a user downloads or forks this example to run it locally, they need to simply rename that file to `auth0-variables.ts` and replace the placeholder values inside the file with the value for their own Auth0 configuration.

When we (as Auth0 employees) work on these example applications, we obviously also need to do this, but we need to take care not to actually check that file into source control. 

Well, that easy you'll say. Just add the file to `.gitignore`.

Yes, that will solve the issue, but it would also create a potential issue for the customers, because it means their own copy of this file will never get checked into source control. Unless of course they remove the entry from the `.gitignore` file.

Well it turns out there is another way. 

In Git, you can exclude a file locally - i.e. only on the current computer.

Inside the `.git` folder for your project, there is a file `/info/exclude`. This file has the exact same structure as a `.gitignore` file, so you can add the file patterns for any files which must be excluded locally, inside that file.

Here is the relevant section from the [Git documentation](https://git-scm.com/docs/gitignore):

> Patterns which are specific to a particular repository but which do not need to be shared with other related repositories (e.g., auxiliary files that live inside the repository but are specific to one user’s workflow) should go into the `$GIT_DIR/info/exclude` file.

Hat tip to [this great StackOverflow answer](https://stackoverflow.com/questions/1753070/how-do-i-configure-git-to-ignore-some-files-locally).