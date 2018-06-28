---
title: "Communicate the status of a background job with SignalR"
description:
  Demonstrates how you can use Coravel to schedule background jobs and then report on the status of a job with SignalR
tags:
- asp.net core
- signalr
- coravel
---

## Introduction

I was looking for a simple [SignalR](https://docs.microsoft.com/en-us/aspnet/core/signalr) sample application that demonstrates how to report the progress of a background job, but I was unable to find anything. About three years ago I wrote a blog post explaining how you can [communicate from an Azure WebJob to your website with SignalR](/blog/communicate-from-azure-webjob-with-signalr), so I decided, since I was unable to find something for the modern versions of ASP.NET Core and SignalR, to write a new version of that blog post.

_That's what the rest of this blog post is about._

In trying to keep things simple, I decided to not use Azure Functions (the modern version of WebJobs) but rather to do everything in one single application.

## Triggering a background job in ASP.NET Core

There are various ways to handle the scheduling of background jobs in ASP.NET Core. One of the more popular ways to do this is to use a library like [Hangfire](https://www.hangfire.io/). For this blog post, I decided to try out a new library called [Coravel](https://github.com/jamesmh/coravel).

I started with a regular ASP.NET Core MVC application and added the Coravel package:

```text
dotnet add package Coravel
```

The register to Coravel queueing services in your `Startup` class:

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        //...

        services.AddQueue();
    }
}
```

To queue a background job, you can inject an `IQueue` instance into your controller and call the `QueueAsyncTask` method with the code for your background task. In the sample code below I queue a background job which simulates incrementing the progress of the job every 100 milliseconds and then writes the progress to the Debug Output window:

```csharp
public class HomeController : Controller
{
    private readonly IQueue _queue;

    public HomeController(IQueue queue)
    {
        _queue = queue;
    }

    public IActionResult Index()
    {
        _queue.QueueAsyncTask(async () =>
        {
            for (int i = 0; i <= 100; i += 5)
            {
                Debug.WriteLine($"Background job progress: {i}");

                await Task.Delay(100);
            }
        });
        return View();
    }
}
```

When you run the application you will notice the following output (_I am using [JetBrains Rider](https://www.jetbrains.com/rider) in the screenshot below_):

![Debug output when running application](/images/blog/2018-06-28-communicate-status-background-job-signalr/debug-output.png)

**Please note** that the Coravel queued jobs will only execute every 30 seconds, so you may have to wait a while for the output to appear.

## Implementing a user flow for the progress

The user flow for the application will be something along the following lines:

![User Flow](/images/blog/2018-06-28-communicate-status-background-job-signalr/user-flow-diagram.png)

To queue a background job, the user can click on the **Queue Background Job** button. This will post back to a `StartProgress` controller action which will queue the job. After the job is queued, the user will be redirected to a **Job Progress** page where they can see the process for the background job.

To implement this flow I have updated my `HomeController` as follows:

```csharp
public class HomeController : Controller
{
    private readonly IQueue _queue;

    public HomeController(IQueue queue)
    {
        _queue = queue;
    }

    public IActionResult Index()
    {
        return View();
    }

    [HttpPost]
    public IActionResult StartProgress()
    {
        string jobId = Guid.NewGuid().ToString("N");
        _queue.QueueAsyncTask(() => PerformBackgroundJob(jobId));

        return RedirectToAction("Progress", new {jobId});
    }

    public IActionResult Progress(string jobId)
    {
        ViewBag.JobId = jobId;

        return View();
    }

    private async Task PerformBackgroundJob(string jobId)
    {
        for (int i = 0; i <= 100; i += 5)
        {
            // TODO: report progress with SignalR

            await Task.Delay(100);
        }
    }
}
```

Note that I create a unique **Job ID** for each job I queue and pass that along when I redirect the user to the `Progress` action.

I have also updated the `Index.cshtml` as follows:

```html
@{
    ViewData["Title"] = "Home Page";
}

<div class="row">
    <div class="col-md-12">
        <h2>Start background process</h2>
        <p>Start a long running process by clicking on the button below:</p>
        <form asp-action="StartProgress">
            <button class="btn btn-primary btn-lg">Queue Background Job</button>
        </form>
    </div>
</div>
```

And added a view for the `Progress` action:

```html
@{
    ViewData["Title"] = "Progress";
}
<h2>@ViewData["Title"]</h2>

<p>Status of your background job: <strong><span id="job-status">Job status will go here...</span></strong></p>
```

## Reporting the progress with SignalR

Right, so let's get to the bit that you are here for: **SignalR**.

I will assume you already know a bit about SignalR. If not you can review the [SignalR documentation](https://docs.microsoft.com/en-us/aspnet/core/signalr) on how to get started.

The first thing I did was to register the SignalR services in my `Startup` class:

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        //...

        services.AddQueue();

        services.AddSignalR();
    }
}
```

I then created a new SignalR hub called `JobProgressHub`:

```csharp
public class JobProgressHub : Hub
{
    public async Task AssociateJob(string jobId)
    {
        Context.Items.Add("JobId", jobId);

        await Groups.AddToGroupAsync(Context.ConnectionId, jobId);
    }

    public override async Task OnDisconnectedAsync(Exception exception)
    {
        if (Context.Items["JobId"] is string jobId)
        {
            await Groups.RemoveFromGroupAsync(Context.ConnectionId, jobId);
        }

        await base.OnDisconnectedAsync(exception);
    }
}
```

When I report the status from my background job using SignalR, I only want to send the status update to the user who initiated the background job. The way I handle this is to create a SignalR Group with the name of the **Job ID**. I added an `AssociateJob` method to my Hub which will handle this.

I also add the **Job ID** to the Context items and then on the `OnDisconnectedAsync` method I check for that **Job ID** and remove the user from the group.

This way, when I report progress, I will just broadcast the progress message to that particular group.

You will also need to register a route for the hub. You can do this by calling the `UseSignalR` method in the `Configure` method of your `Startup` class and specifying a `/jobprogress` route for the hub we just created:

```csharp
public class Startup
{
    public void Configure(IApplicationBuilder app, IHostingEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }
        else
        {
            app.UseExceptionHandler("/Home/Error");
            app.UseHsts();
        }

        app.UseHttpsRedirection();
        app.UseStaticFiles();
        app.UseCookiePolicy();

        app.UseSignalR(routes => { routes.MapHub<JobProgressHub>("/jobprogress"); });
        
        app.UseMvc(routes =>
        {
            routes.MapRoute(
                name: "default",
                template: "{controller=Home}/{action=Index}/{id?}");
        });
    }
}
```

The next thing to do is to update the code for the background job in the `HomeController` to send the message to the group for that **Job ID**. Inject an instance of `IHubContext<JobProgressHub>` into the controller and then call `SendAsync` for the group. 

```csharp
public class HomeController : Controller
{
    private readonly IQueue _queue;
    private readonly IHubContext<JobProgressHub> _hubContext;

    public HomeController(IQueue queue, IHubContext<JobProgressHub> hubContext)
    {
        _queue = queue;
        _hubContext = hubContext;
    }

    private async Task PerformBackgroundJob(string jobId)
    {
        for (int i = 0; i <= 100; i += 5)
        {
            await _hubContext.Clients.Group(jobId).SendAsync("progress", i);

            await Task.Delay(100);
        }
    }
}
```

Note that I specified the name of the method as **progress**. When we create the JavaScript client, we will need to use that same name when listening for events.

OK, so the final piece of the puzzle is to create the JavaScript client. Head back to your `Progress.cshtml` file and update it as follows:

```html
@{
    ViewData["Title"] = "Progress";
}
<h2>@ViewData["Title"]</h2>

<p>Status of your background job: <strong><span id="job-status">Waiting to start...</span></strong></p>

@section Scripts
{
    <script src="~/lib/signalr/signalr.js"></script>
    <script>
        var connection = new signalR.HubConnectionBuilder()
            .withUrl("/jobprogress")
            .configureLogging(signalR.LogLevel.Information)
            .build();
        connection.on("progress",
            (percent) => {
                if (percent === 100) {
                    document.getElementById("job-status").innerText = "Finished!";
                } else {
                    document.getElementById("job-status").innerText = `${percent}%`;
                }
            });
        connection.start()
            .then(_ => connection.invoke("AssociateJob", "@ViewBag.JobId"))
            .catch(err => console.error(err.toString()));
    </script>
}
```

The code above creates a new connection on the `/jobprogress` URL, which is connected to our `JobProgressHub`. I also listed for that **progress** event to update the user interface with the status of the job. The final thing I do is to call the `AssociateJob` method on my `JobProgressHub` to associate this connection with the **Job ID**.

## Testing it out

The animation below demonstrates the application in action.

![The application in action](/images/blog/2018-06-28-communicate-status-background-job-signalr/in-action.gif)

## Next steps

This demo application is certainly not what I would call _production ready_. For one thing, I do not persist any information about the background jobs. When I redirect the user to the Progress page I simply assume we are waiting for the job to start. But it may be that the job has already completed a long time ago. Or that no job with that **Job ID** exists.

You get the idea. This is not a perfect application. It merely demonstrates how you can use SignalR for something other than your typical chat application. Feel free to use the techniques I described here as you please.

Source code is available at [https://github.com/jerriepelser-blog/signalr-long-running](https://github.com/jerriepelser-blog/signalr-long-running).