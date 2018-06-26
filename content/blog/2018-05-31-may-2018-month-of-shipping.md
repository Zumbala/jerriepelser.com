---
title: "May 2018: My month of shipping"
description:
  I dedicated May to finishing and shipping all of the half-baked ideas that have been brewing for the past years. This is how it went.
tags:
- general
---

When I resigned from Auth0 at the end of last year, it was to work my own ideas. Sadly, six months in, I had very little to show for it. The only thing of significance I managed to ship was the [Airport Explorer Book](/books/airport-explorer), but nothing other than that.

I had a bunch of ideas - some on which I started coding and others just brewing in my head - but I would jump around between them and never finished anything. I began to grow a little bit disgusted with myself about the inability to focus and finish any of these projects.

At the end of April, I finally had enough and decided to dedicate the month of May to finish all these projects. I made a list of the outstanding projects I wanted to complete and that I would devote myself to during this time.

Here is what I shipped...

## GitHub Issues CLI

My first project to ship was the [GitHub Issues CLI](https://github.com/jerriep/github-issues-cli). This one had been in development for a few weeks, but there was not much focus on driving it to completion. Once I got my head down and focused, I finished all the outstanding features within a day. A few nice-to-have features are missing, but it can do all the most common tasks such as listing issues, viewing an issue, creating a new issue, as well as closing and re-opening issues.

![GitHub Issues CLI Screenshot](/images/blog/2018-05-31-may-2018-month-of-shipping/github-issues-cli.png)

## Git Status CLI

Next on the list was the [Git Status CLI](https://github.com/jerriep/git-status-cli). I do work on many Git repositories, and sometimes I forget to commit changes or to push commits to GitHub. You can use the `git status` command to get the status of the working tree, but that only works on the current repository.

Git Status CLI scans a directory, along with its sub-directories, to find Git repositories, and reports on the status of all of those repositories.

![Git Status CLI Screenshot](/images/blog/2018-05-31-may-2018-month-of-shipping/git-status-cli.png)

Yep, there are scripts that can do this for you, but since when has that ever stopped a programmer from thinking they can do it "better" 😏

## .NET Jobs

There are many job websites out there, but all of them are pretty generic. By that, I mean that they cover a wide range of industries, or, in the case of dedicated programming job boards, they cover many different programming languages. This can make it difficult to find jobs which match your set of requirements for technologies.

Let's say, for example, that you want to find all ASP.NET Core jobs which also require Entity Framework and Angular. Going to a job board and searching for "asp.net core entity framework angular" will not give you what you are looking for.

So, I set about creating a dedicated .NET Job board with a deep understanding of the related technologies. I scrape other job boards and analyse the job descriptions to extract all the technologies referenced in a job posting. I then built a proper search interface across all of this using [Algolia](https://www.algolia.com/), and the result is [.NET Jobs](https://www.finddotnetjobs.com/).

It lets you search for jobs based on keywords and then allow you to filter them based on the combination of technologies mentioned in the job posting.

![.NET Jobs Screenshot](/images/blog/2018-05-31-may-2018-month-of-shipping/dotnet-jobs.png)

## dotnet-outdated

The last one I tackled was [dotnet-outdated](https://github.com/jerriep/dotnet-outdated). When using an IDE such as Visual Studio, it is easy to find out whether newer versions of the NuGet packages used by your project is available, by using the NuGet Package Manager. However, the .NET Core command-line tools do not provide a built-in way for you to report on outdated NuGet packages.

**dotnet-outdated** is a .NET Core Global tool that allows you to quickly report on any outdated NuGet packages in your .NET Core and .NET Standard projects.

![dotnet-outdated Screenshot](/images/blog/2018-05-31-may-2018-month-of-shipping/dotnet-outdated.png)

## What I learned

Firstly, none of these tools I developed is earth-shattering, but it was good for me to get back into the habit of shipping. For about six months I have not shipped anything, so to release four products - albeit small ones - in one month, is a massive positive step forward.

I enjoyed working on the three command-line utilities a lot, but the work on .NET Jobs was not too enjoyable. It had some interesting bits, but overall it just does not inspire me.

When I created the plan for the month, I had ideas for a few more web projects, but after .NET Jobs, I canned those as I came to the realisation that I do not enjoy web development. The overall ecosystem has just become so complicated and developing a web app feels like a slow and painful process. Web apps are much more powerful these days, but in a way, I long for the days when doing web development was so much simpler.

Thinking about it, I am not sure that I ever really enjoyed web development that much. I need to explore that thought a bit more.

I am also not sure what that means for my career, as web development is the future. For me, I feel that God has always lead me down the right path, so I will have faith and keep exploring this path and see where it leads me.

## Up next in June

So what is happening in June? I have not created a definite plan yet, as I wanted to take a moment to bask in the glory of shipping something 😉 However, my preliminary ideas for June are as follows:

1. Updates to the three existing command-line utilities. I have a long list of issues I want to address - some bugs that need fixing and requests for enhancements from users.
1. I have an idea for another app I will personally find useful. The one will be a UWP app, actually 😱
1. I also want to contribute more to the docs for Nate McMaster's [CommandLineUtils](https://github.com/natemcmaster/CommandLineUtils) which has been tremendously useful.
1. Depending on time, I also have ideas for a few other utilities. Maybe I can start on some of those.
1. If time allows, there were a few exciting things I learned in the process which I can turn into blog posts.