---
title: "Determine the GitHub repository from the working directory"
description:
  Determine whether the current working directory is part of a Git repository, and if so, retrieve information for the remote GitHub repository.
date: 2018-04-11
tags:
- dotnet
- dotnet core
- cli
- github
- git
url: /blog/determine-github-repo-from-current-working-directory
---

Lately, I have been playing quite a bit with console applications. There are a couple of reasons for this. Firstly, I want to get a better understanding of developing [.NET Core Global tools](https://blogs.msdn.microsoft.com/dotnet/2018/02/02/net-core-2-1-roadmap/). 

Secondly, I have grown pretty tired of web applications and the craziness and constant turmoil which is the JavaScript landscape, so I decided to go back to basics: developing simple console applications. Yep, that's right. While the cool kids are going crazy over Angular, React, Webpack and who knows what else, I have decided to go the opposite direction.

The first project I am tackling is a [CLI for listing and managing GitHub issues from the command line](https://github.com/jerriep/github-issues-cli). I have no idea if there is a need for something like this, but it sounded like an interesting problem, and that is my primary motivation at the moment: to solve interesting problems.

## The problem

The GitHub API has an endpoint which allows you to [retrieve all issues for the authenticated user](https://developer.github.com/v3/issues/#list-issues), and that works great in my application:

![Listing all issues for the authenticated user](/images/blog/2018-04-11-determine-github-repo-from-current-working-directory/listing-issues-for-user.png)

But, when a user is running the CLI from a folder which is part of a Git repository, and that Git repository is hosted on GitHub, it makes more sense to list the issues for that GitHub repository rather than for all repositories.

## Get the GitHub remote for the current directory

The first issue I needed to solve was how to determine whether the current folder is part of a Git repository. Thankfully, the [libgit2csharp project](https://github.com/libgit2/libgit2sharp/) came to my rescue as it has all the necessary functionality for this.

Ensure you install the NuGet package:

```text
dotnet add package LibGit2Sharp
```

Then, you can call the `Discover` method of the `Repository` class, passing the directory:

```csharp
string repoPath = LibGit2Sharp.Repository.Discover(System.Environment.CurrentDirectory);
```

If the directory you passed is part of a GitHub repository, it will return the path to the directory for the Git repo. Otherwise, it will return null.

So for example, if the current directory is `C:\Development\jerriep\github-issues-cli\src\GitHubIssuesCli`, it will return something like `C:\Development\jerriep\github-issues-cli\.git\`.

Now that you have the path to the repo, you can create a new instance of the `Repository` class, passing that path:

```csharp
var repository = new LibGit2Sharp.Repository(repoPath);
```

The new `Repository` instance has lots of information about your Git repo, one of which is the [list of remotes](https://git-scm.com/book/en/v2/Git-Basics-Working-with-Remotes). We can query those to look for the remote GitHub repo.

In the sample below I just grabbed the one named `origin`, but you should probably do something more fault tolerant. For one thing, it may not be called `remote`, and also there may be multiple remotes, and not all of them are hosted on GitHub. 

```csharp
var remote = repository.Network.Remotes.FirstOrDefault(r => r.Name == "origin");

string remoteUrl = remote.Url;
```

## Clean up the URL

Okay, so now we have the URL to the remote. In my case, this is `https://github.com/jerriep/github-issues-cli.git` because I am using HTTPS. But I could also have used SSH, and in that case, it would have been `git@github.com:jerriep/github-issues-cli.git`.

I want to use the `Uri` class to parse the URL, but if I pass it a URL like `git@github.com:jerriep/github-issues-cli.git` to the `Uri` constructor, it will throw a `UriFormatException` stating that the URI scheme is not valid.

So, what I decided to do is to clean up a URL that starts with `git@...` to make it look like an `https://...` URL. I also want to remove the `.git` at the end.

```csharp
// We need to do some normalization on the URL
// 1. Remove .git at the end
remoteUrl = Regex.Replace(remoteUrl, @"\.git$", "");

// 2. Normalize git@ and https:git@ urls
remoteUrl = Regex.Replace(remoteUrl, "^git@", "https://");
remoteUrl = Regex.Replace(remoteUrl, "^https:git@", "https://");
remoteUrl = Regex.Replace(remoteUrl, ".com:", ".com/");
```

With that done, I will end up with a nice looking URL using the HTTPS scheme which the `Uri` class understands.

```csharp
Uri remoteUri = new Uri(remoteUrl);
```

## Get the user and repo

All that remains is to look at the `AbsolutePath` of the URI which, in my case, will be `jerriep/github-issues-cli` and split it at the `\` which will give me the owner and the repository name.

```csharp
// Now let's split the path
string[] pathSegments = remoteUri.AbsolutePath.Split('/', StringSplitOptions.RemoveEmptyEntries);

// Now we should be left with (at least) 2 entries, containing the user and the repo
if (pathSegments != null && pathSegments.Length >= 2)
{
    string owner = pathSegments[0];
    string repo = pathSegments[1];
}
```

## Retrieve the issues for the repo

I have everything in place to call the [endpoint to list issues for a repository](https://developer.github.com/v3/issues/#list-issues-for-a-repository). I'll be making use of [octokit.net](https://github.com/octokit/octokit.net), so here's what that code will look like:

```csharp
var gitHubClient = new GitHubClient(new ProductHeaderValue("MyAmazingApp"))
{
    Credentials = new Credentials("your github token")
};

var issues = await gitHubClient.Issue.GetAllForRepository(owner, repo);
```

And with that in place, when I run my CLI from a folder which has a GitHub remote, and that remote contains issues, I will see the issues displayed on the screen:

![Listing all issues for the current repo](/images/blog/2018-04-11-determine-github-repo-from-current-working-directory/list-issues-for-repo.png)

BTW, in the screenshot above, my app lists all the issues for the current repo. I should only display the ones for the current user in that repo. That is a bug is still need to fix.

## Source code

You can have a look at the [source code for my GitHub Issues CLI](https://github.com/jerriep/github-issues-cli) if you want to see more details of how this application is implemented.

Also, at the time of writing this blog post on 11 April 2018, the actual CLI is not yet available to be installed. I will publish it at a later date once it is finished.