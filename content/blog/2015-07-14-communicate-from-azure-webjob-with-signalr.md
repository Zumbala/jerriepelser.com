---
date: 2015-07-14T00:00:00Z
description: |
  Demonstrates how you can use SignalR to communicate the progress of background tasks from a Azure WebJob to your ASP.NET website.
tags:
- aspnet
- aspnet mvc
- signalr
- azure webjobs
title: Communicate from an Azure WebJob to your website with SignalR
url: /blog/communicate-from-azure-webjob-with-signalr/
---

## Introduction 

Azure WebJobs are great for offloading long running tasks from you website, but sometimes you find that you may want to communicate the progress of those tasks back to the user. In this blog post I will demonstrate how you can communicate progress from an Azure WebJob back to the browser using SignalR.

It is important to understand that the SignalR connection is made between the web server running the ASP.NET Web application and the browser. This means that in order for the Azure WebJob to communicate progress to the browser it would need to request the ASP.NET Web application to do so.
 
As an example I am going to create an application where a user can create an arbitrary job from an ASP.NET Website. When a job is submitted it is placed on an Azure Queue where an Azure WebJob will monitor the queue and process the job. To simulate the processing of a job I will just increment a percentage from 0 to 100 with a bit of a delay inbetween progress to simulate time which is elapsing.

Also, after a user has submitted a job they will be redirected to a progress screen. As the WebJob progresses through the job it will call a URL on the ASP.NET website which will in turn communicate back to the browser via SignalR to update the actual progress.

This blog post is going to assume you have some basic working knowledge of both Azure WebJobs as well as SignalR, and is not intended as a step-by-step guide.

## Submitting a job

For the demo project I created a Solution with two projects. The one project is an ASP.NET Website and the other an Azure WebJob.

To submit a job I created a page where a user can click on a button create a new Job. 

![](/assets/images/communicate-from-azure-webjob-with-signalr/home-page.png)

Once the user clicks the "Start a New Job" button, the application creates a new message on the queue which will contain an arbitrary Job ID. After the message has been created the user is redirected to a "Job Status" page where they can then monitor the status of that particular job.

``` csharp
public async Task<ActionResult> CreateJob()
{
    var jobId = Guid.NewGuid();

    var storageAccount = CloudStorageAccount.Parse(ConfigurationManager.ConnectionStrings["QueueStorageConnectionString"].ConnectionString);
    var cloudQueueClient = storageAccount.CreateCloudQueueClient();

    var queue = cloudQueueClient.GetQueueReference("jobqueue");
    queue.CreateIfNotExists();

    CloudQueueMessage message = new CloudQueueMessage(JsonConvert.SerializeObject(jobId));
    await queue.AddMessageAsync(message);

    return RedirectToAction("JobStatus", new {jobId});
}
```

For the Job Status page I have a very basic controller action which creates an instance of a `JobStatusViewModel` class that contains the `JobId`, and renders the status page:

``` csharp
public ActionResult JobStatus(Guid jobId)
{
    var viewModel = new JobStatusViewModel
    {
        JobId = jobId
    };

    return View(viewModel);
}
``` 

The actual view itself is also very basic. It displays a title, as well as a progress. Currently the progress does nothing, but once we add the SignalR communication we will be updating the `span` element with the current progress of the job:

``` html
@model Website.Models.JobStatusViewModel

<h2>Status for Job @Model.JobId</h2>
<p>Current progress: <span id="progress-span">0</span></p>
```

## Processing the Job

The Azure WebJob which processes the Job is very simple. It simply loops from 10 to 100, incrementing by 10 each time and I have also added a delay to simlate a job which takes a while to complete. Also on each iteration of the loop it calls a `CommunicateProgress` method which simply calls back to the notification endpoint in the ASP.NET Web application, passing along the Job ID as well as the progress for that particular job.

``` csharp
public class Functions
{
    public static async Task ProcessQueueMessage([QueueTrigger("jobqueue")] Guid jobId, TextWriter log)
    {
        for (int i = 10; i <= 100; i+=10)
        {
            Thread.Sleep(400);

            await CommunicateProgress(jobId, i);
        }
    }

    private static async Task CommunicateProgress(Guid jobId, int percentage)
    {
        var httpClient = new HttpClient();

        var queryString = String.Format("?jobId={0}&progress={1}", jobId, percentage);
        var request = ConfigurationManager.AppSettings["ProgressNotificationEndpoint"] + queryString;

        await httpClient.GetAsync(request);
    }
}
```

In the `app.config` for my WebJob I have configured the `ProgressNotificationEndpoint` value to point to `http://localhost:49728/Home/ProgressNotification` which is a URL in the same ASP.NET Web application that created the job. The actual implementation for this method will be done later in the blog post.

``` xml
<appSettings>
	<add key="ProgressNotificationEndpoint" value="http://localhost:49728/Home/ProgressNotification" />
</appSettings>
``` 

## Implementing SignalR

This is not a step-by-step walkthrough of SignalR. I assume you have knowledge of SignalR, so if you have not used it before please work through the [Getting Started with SignalR 2 and MVC 5](http://www.asp.net/signalr/overview/getting-started/tutorial-getting-started-with-signalr-and-mvc) tutorial first.

One trick with SignalR is that I want to map the actual SignalR connections to the Job IDs. So when a SignalR connection is opened from the browser I want to also pass along the Job ID that that connection is listening for progress for. This is so that when the server wants to communicate progress back for a specific Job, I simply want to get a list of connections which are listening for that job and only communicate the progress for that Job back to them.  

Luckily the ASP.NET website has once again come to my rescue with a solution for this. I am using the same principles (and some code) used by Tom FitzMacken in the tutorial [Mapping SignalR Users to Connections](http://www.asp.net/signalr/overview/guide-to-the-api/mapping-users-to-connections).

You can read the article above in detail but basically I create a `ConnectionMapping` class which will associate SignalR connections with an arbitrary key value (in this case the Job ID):

``` csharp
public class ConnectionMapping<T>
{
    private readonly Dictionary<T, HashSet<string>> connectionStore =
        new Dictionary<T, HashSet<string>>();

    public int Count
    {
        get { return connectionStore.Count; }
    }

    public void Add(T key, string connectionId)
    {
        lock (connectionStore)
        {
            HashSet<string> connections;
            if (!connectionStore.TryGetValue(key, out connections))
            {
                connections = new HashSet<string>();
                connectionStore.Add(key, connections);
            }

            lock (connections)
            {
                connections.Add(connectionId);
            }
        }
    }

    public IEnumerable<string> GetConnections(T key)
    {
        lock (connectionStore)
        {
            HashSet<string> connections;
            if (connectionStore.TryGetValue(key, out connections))
            {
                return connections;
            }
        }

        return Enumerable.Empty<string>();
    }

    public void Remove(T key, string connectionId)
    {
        lock (connectionStore)
        {
            HashSet<string> connections;
            if (!connectionStore.TryGetValue(key, out connections))
            {
                return;
            }

            lock (connections)
            {
                connections.Remove(connectionId);

                if (connections.Count == 0)
                {
                    connectionStore.Remove(key);
                }
            }
        }
    }
}
```

When a connection is established from the browser I will also pass along a query string called "jobId" to the hub which will contain the actual Job ID. Then in my SignalR Hub I monitor the connection lifetime events and associate or disassociate that particular connection with the Job ID that was passed in the query string:

``` csharp
public class JobProgressHub : Hub
{
    private readonly static ConnectionMapping<Guid> Connections = new ConnectionMapping<Guid>();

    public override Task OnConnected()
    {
        Guid jobid = GetJobId();

        Connections.Add(jobid, Context.ConnectionId);

        return base.OnConnected();
    }

    public override Task OnDisconnected(bool stopCalled)
    {
        Guid jobid = GetJobId();

        Connections.Remove(jobid, Context.ConnectionId);

        return base.OnDisconnected(stopCalled);
    }

    public override Task OnReconnected()
    {
        Guid jobId = GetJobId();

        if (!Connections.GetConnections(jobId).Contains(Context.ConnectionId))
        {
            Connections.Add(jobId, Context.ConnectionId);
        }

        return base.OnReconnected();
    }

    private Guid GetJobId()
    {
        return new Guid(Context.QueryString["jobId"]);
    }

    public static IEnumerable<string> GetUserConnections(Guid jobId)
    {
        return Connections.GetConnections(jobId);
    }
}
```

Next up we need to create the controller action which is called by the Azure Web Job in its `CommunicateProgress` method. This action takes a `jobId` and `progress` as parameters. It will then retrieve the SignalR connections which are listening for progress on that particular job and call the `updateProgress` method on the client-side of the connection, passing along the actual progress:

``` csharp
public ActionResult ProgressNotification(Guid jobId, int progress)
{
    var connections = JobProgressHub.GetUserConnections(jobId);

    if (connections != null)
    {
        foreach (var connection in connections)
        {
            // Notify the client to refresh the list of connections
            var hubContext = GlobalHost.ConnectionManager.GetHubContext<JobProgressHub>();
            hubContext.Clients.Clients(new[] { connection }).updateProgress(progress);
        }
    }

    return new HttpStatusCodeResult(HttpStatusCode.OK);
}
```

The last bit is to implement SignalR in the Job Status Razor view:

``` html
@model Website.Models.JobStatusViewModel

<h2>Status for Job @Model.JobId</h2>
<p>Current progress: <span id="progress-span">0</span></p>

@section scripts {
    @Scripts.Render("~/bundles/jquery")
    <script src="~/Scripts/jquery.signalR-2.2.0.min.js"></script>
    <script src="~/signalr/hubs"></script>

    <script type="text/javascript">
        $(document).ready(function() {
            var jobId = "@Model.JobId";

            // Reference the auto-generated proxy for the hub.
            var jobProgressHub = $.connection.jobProgressHub;

            // Create a function that the hub can call back to display progress
            jobProgressHub.client.updateProgress = function(progress) {
                $("#progress-span").text(progress);

                console.log("Progress: " + progress);
            };

            $.connection.hub.logging = true;
            $.connection.hub.qs = "jobId=" + jobId;
            $.connection.hub.start();
        });
    </script>
}
``` 

The scripts section in the view does the following:

1. First I added references to the JQuery and SignalR scripts
2. Then I assign the Job ID (which is contained in the Model for the view) to the `jobId` variable. (This happens when the Razor view is rendered)
3. I reference the proxy for my SignalR Hub
4. I create the `updateProgress` method implementation which simply updates the `span` element with the current progress
5. Lastly I set the `jobId` query string parameter for the SignalR connection and start the connection

Here is short video of the final version in action. Notice that as soon as the WebJob picks up the new job on the queue, you can see the current progress in the browser starting to increment: 

<iframe width="640" height="360" src="https://www.youtube.com/embed/HV5WPj0w6E8" frameborder="0" allowfullscreen></iframe>

## Conclusion

In this blog post I showed you how you can implement progress reporting from an Azure WebJob by making a callback to the originating web application and using SignalR to communicate the progress back from the webserver to the browser.
