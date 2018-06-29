---
title: "Change Favicon based on the application state"
description:
  Changing your application's favicon is a convenient way to communicate the current state of the application to the user.
tags:
- asp.net core
---

## Introduction

GitHub has a great feature which communicates the status of a Pull Request by changing the Favicon of the page. In the screenshots below you can see examples of this.

When all checks have not yet completed:

![All checks have not yet completed](/images/blog/2018-06-29-change-favicon-application-state/checks-not-completed.png)

When all checks have passed:

![All checks have passed](/images/blog/2018-06-29-change-favicon-application-state/all-checks-passed.png)

When some checks have failed:

![Some checks have failed](/images/blog/2018-06-29-change-favicon-application-state/some-checks-failed.png)

My blog post from yesterday which demonstrates [how to communicate the status of a background job with SignalR](/blog/communicate-status-background-job-signalr/) is a great candidate for something like this. In this blog post, I will show how you can implement something similar to GitHub.

## Creating status Favicons

First, I needed to find or create Favicons I can use to reflect the status of a background job. [Font Awesome](https://fontawesome.com/) contains many icons which fit the bill, but first I needed to convert them into a format I can use as a Favicon.

I found the [Font Awesome Favicon Generator](https://gauger.io/fonticon/) which allowed me to do just this.  It is as simple as finding an appropriate icon and downloading it as a PNG:

![Font Awesome Favicon Generator](/images/blog/2018-06-29-change-favicon-application-state/favicon-generator.png)

## Changing the Favicon based on the status

To change the Favicon for a web page, I came across [this Gist](https://gist.github.com/mathiasbynens/428626) which demonstrates how to do it.

I updated the code from yesterday's blog to also update the Favicon upon status change. The Favicon defaults to the _waiting_ status. Once some progress is received, I change it to a _running_ status and then finally, when the background job is complete, I change it to a _done_ status:

```js
document.head || (document.head = document.getElementsByTagName('head')[0]);

var connection = new signalR.HubConnectionBuilder()
    .withUrl("/jobprogress")
    .configureLogging(signalR.LogLevel.Information)
    .build();
connection.on("progress",
    (percent) => {
        if (percent === 100) {
            changeFavicon("/done.png");
            document.getElementById("job-status").innerText = "Finished!";
        } else {
            changeFavicon("/running.png");
            document.getElementById("job-status").innerText = `${percent}%`;
        }
    });
connection.start()
    .then(_ => connection.invoke("AssociateJob", "@ViewBag.JobId"))
    .catch(err => console.error(err.toString()));

changeFavicon("/waiting.png");

function changeFavicon(src) {
    var link = document.createElement('link'),
        oldLink = document.getElementById('dynamic-favicon');
    link.id = 'dynamic-favicon';
    link.rel = 'shortcut icon';
    link.href = src;
    if (oldLink) {
        document.head.removeChild(oldLink);
    }
    document.head.appendChild(link);
```

Here are some screenshots of the updated application in action, demonstrating each of these states. Note the different Favicon in each screenshot.

![Background job is waiting](/images/blog/2018-06-29-change-favicon-application-state/job-waiting.png)

![Background job is running](/images/blog/2018-06-29-change-favicon-application-state/job-running.png)

![Background job is complete](/images/blog/2018-06-29-change-favicon-application-state/job-done.png)