---
title: "Updating your .NET project dependencies with Dependabot"
description:
  Dependabot is a service that will automatically create pull requests to update outdated dependencies in your .NET projects.
tags:
- .net
- .net core
- dependabot
- nuget
---

## Introduction

Back in May, [I developed a .NET Core Global tool](/blog/may-2018-month-of-shipping/) called [dotnet-outdated](https://github.com/jerriep/dotnet-outdated) that allows you to check for outdated dependencies (NuGet packages) in your project and automatically update those. This is great, but it requires you to you run it manually.

I recently came across a service called [Dependabot](https://dependabot.com/) that will automatically run checks against your GitHub repository to check for any outdated dependencies and create Pull Requests to update those dependencies.

I decided to give it a try on the **dotnet-outdated** repository. Here is a walk-through of this process with some screenshots.

## Sign up for Dependabot

You will need to sign up for Dependabot, so head over to the [Dependabot home page](https://dependabot.com/) and click the **Sign up** button. Then, select to **Sign up with GitHub**.

![Sign up for Dependabot](/images/blog/2018-09-12-updating-net-dependencies-dependabot/signup.png)

You will need to give access to GitHub account.

![Give access to your GitHub account](/images/blog/2018-09-12-updating-net-dependencies-dependabot/github-authorize.png)

And install Dependabot on GitHub account

![Install Dependabot on your GitHub account](/images/blog/2018-09-12-updating-net-dependencies-dependabot/install-dependabot-to-github-account.png)

This will take you to the Dependabot page on the GitHub Marketplace. Scroll down to the pricing options, select the **Open source/personal account** option and then click the **Install it for free** button.

![Install Dependabot for free](/images/blog/2018-09-12-updating-net-dependencies-dependabot/install-dependabot-for-free.png)

Review your order and select the **Complete order and begin installation** button.

![Review your GitHub Marketplace order](/images/blog/2018-09-12-updating-net-dependencies-dependabot/review-order.png)

You will be taken to another screen which will allow you to give access to Dependabot to all your repositories or just specific repositories. Select the repositories you want to provide access to and click the **Install** button.

![Give access to repositories](/images/blog/2018-09-12-updating-net-dependencies-dependabot/give-access-to-repositories.png)

## Configure a repository to watch 

You will be redirected back to the Dependabot website where you can now add repositories for Dependabot to watch. Click the **Add Repo** button.

![Add a new repository](/images/blog/2018-09-12-updating-net-dependencies-dependabot/dependabot-repositories.png)

This will open a dialog where you can select the repository and the language. In my case, I selected the **dotnet-outdated** repo and set the language to **.NET**. You can configure _advanced options_ if you want, and once done you can click on the **Add** button.

![Add a new repository](/images/blog/2018-09-12-updating-net-dependencies-dependabot/dependabot-add-repo.png)

After the repo has been added, you will see it displayed in the list of repositories which are configured with Dependabot. The status will indicate that it is busy checking the repository for newer dependencies.

![Repository status](/images/blog/2018-09-12-updating-net-dependencies-dependabot/dependabot-repo-added.png)

Once the repository has been checked, the status will be updated. You can head over to the repository's page on GitHub and head see whether any Pull Requests have been created. 

In my case, you can see that it has created PRs for five dependencies to be updated. Also notice the little yellow lightbulb next to each PR to indicate that checks are running - in this case, my automated tests which have been configured with [AppVeyor](https://www.appveyor.com/).

![List of GitHub PRs](/images/blog/2018-09-12-updating-net-dependencies-dependabot/github-prs.png)

I can open one of the pull requests to view more information about the Pull Request. When I scroll down, I can see that the checks are still running.

![GitHub PR checks still running](/images/blog/2018-09-12-updating-net-dependencies-dependabot/github-pr-checks-running.png)

When I head over to the _Files changed_ tab, I can see that the two project files have been updated and the version number of the `McMaster.Extensions.CommandLineUtils` package has been bumped to the latest version.

![GitHub PR files changed](/images/blog/2018-09-12-updating-net-dependencies-dependabot/github-pr-files-changed.png)

After a while, my AppVeyor build has completed, I can see that all checks have passed.

![GitHub PR checks have passed](/images/blog/2018-09-12-updating-net-dependencies-dependabot/github-pr-checks-completed.png)

At this point, given that my test coverage is good, I can be sure that upgrading this dependency has not caused any issues and I can simply merge the Pull Request.

![GitHub PR merged](/images/blog/2018-09-12-updating-net-dependencies-dependabot/github-pr-merged.png)

Dependabot even cleans up after itself, so if you give it a few seconds you will notice the status updating and stating that Dependabot has deleted the branch it created for the PR.

![GitHub PR branch deleted](/images/blog/2018-09-12-updating-net-dependencies-dependabot/github-pr-branch-deleted.png)

## When things go wrong

Thankfully, since I have configured AppVeyor to run my application's tests when a new PR is submitted, I can catch dependency upgrades which are going to break my application. As you can see in the screenshot below, the upgrade of the `System.IO.Abstractions` package to the latest version causes my build to fail:

![GitHub PR checks failing](/images/blog/2018-09-12-updating-net-dependencies-dependabot/github-pr-checks-failing.png)

This gives me the opportunity to look and the AppVeyor logs to see what the error is that occurred. I can then out that PR locally and make updates to my code.

## Conclusion

Dependabot provides a great, hands-off way to automatically keep your project's dependencies up to date. Dependabot allows for a number of configuration options, such as how often outdated dependencies should be checked, as well as whether to automatically merge PRs if all checks have been passed successfully.