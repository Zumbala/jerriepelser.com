---
title: "Creating a Github Webhook with ASP.NET Core and AWS Lambda"
description: |
  AWS Lambda is an ideal use case for developing GitHub Webhooks. Here's looking at how you can implement one using ASP.NET Core.
date: 2018-01-23
tags:
- .net core
- asp.net
- asp.net core
- aws
- aws lambda
- aws api gateway
url: /blog/create-github-webhook-aspnetcore-aws-lambda/
---

Amazon recently [announced support](https://aws.amazon.com/about-aws/whats-new/2018/01/aws-lambda-supports-c-sharp-dot-net-core-2-0/) for .NET Core 2.0 on AWS Lambda. In this blog post, I will look at how we can use ASP.NET Core 2.0 to create a GitHub webhook using AWS Lambda and AWS API Gateway.

Before we start, ensure that you have the latest version of the [AWS Toolkit for Visual Studio](https://aws.amazon.com/visualstudio/) installed.

## Creating a new ASP.NET Core API

One of the programming models which AWS supports is to host a complete ASP.NET Core application in Lambda. I discussed this previously in [Getting started with .NET Core and AWS Lambda](https://www.jerriepelser.com/blog/getting-started-with-dotnet-core-aws-lambda/) and [Creating a Serverless Application with ASP.NET Core, AWS Lambda and AWS API Gateway](https://www.jerriepelser.com/blog/aspnet-core-aws-lambda-serverless-application/), so feel free to check out those blog posts if you are not familiar with this concept.

In Visual Studio create a new project (File > New > Project) and select the **AWS Serverless Application (.NET Core)** template:

![Create a new project](/assets/images/2018-01-23-create-github-webhook-aspnetcore-aws-lambda/create-new-project.png)

Next, select the **ASP.NET Core Web API** blueprint:

![Select AWS Bluepring](/assets/images/2018-01-23-create-github-webhook-aspnetcore-aws-lambda/select-aws-blueprint.png)

This will create a new ASP.NET Core API with a couple of sample controllers which you can deploy as-is to AWS.

> For the purposes of this blog post, I am not interested in any of the sample controllers so I got rid of it, along with some of the related settings in the `serverless.template` file. This is not necessary, but if you want to do the same, you can have a look at [this commit](https://github.com/jerriepelser-blog/LambdaGitHubWebhook/commit/97d9185683fc33126aeb54211c488c0af1a5546f) to see the changes I made to the default template.

## Creating the webhook

Microsoft has created the [ASP.NET Webhooks](https://github.com/aspnet/aspnetwebhooks) libraries to make consuming webhooks easier, but at the time of writing this blog post the [ASP.NET Core Webhooks](https://github.com/aspnet/WebHooks) libraries have not been released yet, so this means we will need to do some extra code to handle the incoming webhook.

> Most of the code I use to handle the GitHub webhook was taken from the blog post [Validate and Secure GitHub Webhooks In C# With ASP.NET Core MVC](http://michaco.net/blog/HowToValidateGitHubWebhooksInCSharpWithASPNETCoreMVC) written by _Michael Conrad_, so hat tip to him.

Create a new controller called `GithubWebhookController` with a constructor which takes instances of `IConfiguration` and `ILogger<GithubWebhookController>` (the [ASP.NET Core Dependency Injection](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection) will take care of injecting the correct instances of these at runtime).

```csharp
[Route("webhooks/github")]
public class GithubWebhookController : Controller
{
    private const string Sha1Prefix = "sha1=";
    private readonly string _gitHubWebhookSecret;
    private readonly ILogger<GithubWebhookController> _logger;

    public GithubWebhookController(IConfiguration configuration, ILogger<GithubWebhookController> logger)
    {
        _logger = logger;

        _gitHubWebhookSecret = configuration["GitHubWebhookSecret"];
    }
}
```

Notice that we are reading a configuration variable called `GitHubWebhookSecret`. This is required for us to [secure the GitHub webhook](https://developer.github.com/webhooks/securing/) and is used by the GitHub signature validation function.  Also, notice that the route for the controller is `webhooks/github`.

Next, create a couple of functions which will handle the actual signature validation. The code for these functions was taken from the blog post by _Michael Conrad_ I linked to earlier, and adapted slightly.

This function computes a hash (using the secret as the key) from the payload that the GitHub is sending and validating that it corresponds with the value which GitHub will pass along in an `X-Hub-Signature` header.

``` csharp
[Route("webhooks/github")]
public class GithubWebhookController : Controller
{
    private const string Sha1Prefix = "sha1=";
    private readonly string _gitHubWebhookSecret;
    private readonly ILogger<GithubWebhookController> _logger;

    public GithubWebhookController(IConfiguration configuration, ILogger<GithubWebhookController> logger)
    {
        _logger = logger;

        _gitHubWebhookSecret = configuration["GitHubWebhookSecret"];
    }

    private bool IsGitHubSignatureValid(string payload, string signatureWithPrefix)
    {
        if (string.IsNullOrWhiteSpace(payload))
            throw new ArgumentNullException(nameof(payload));
        if (string.IsNullOrWhiteSpace(signatureWithPrefix))
            throw new ArgumentNullException(nameof(signatureWithPrefix));

        if (signatureWithPrefix.StartsWith(Sha1Prefix, StringComparison.OrdinalIgnoreCase))
        {
            var signature = signatureWithPrefix.Substring(Sha1Prefix.Length);
            var secret = Encoding.ASCII.GetBytes(_gitHubWebhookSecret);
            var payloadBytes = Encoding.ASCII.GetBytes(payload);

            using (var hmacsha1 = new HMACSHA1(secret))
            {
                var hash = hmacsha1.ComputeHash(payloadBytes);

                var hashString = ToHexString(hash);

                if (hashString.Equals(signature))
                    return true;
            }
        }

        return false;
    }

    public static string ToHexString(byte[] bytes)
    {
        StringBuilder builder = new StringBuilder(bytes.Length * 2);
        foreach (byte b in bytes)
        {
            builder.AppendFormat("{0:x2}", b);
        }

        return builder.ToString();
    }
}
```

The final part is to write the actual controller action. This action extracts the value from the `X-Hub-Signature` header, as well as reading the actual payload, and then passing that along to our `IsGitHubSignatureValid` method to ensure that the request was initiated by GitHub and not sent by some 3rd party.

Notice that we are also logging some other values received from GitHub. These will show up in the CloudWatch logs at runtime.

```csharp
[Route("webhooks/github")]
public class GithubWebhookController : Controller
{
    // Some code omitted for brevity...

    [HttpPost("")]
    public async Task<IActionResult> Receive()
    {
        Request.Headers.TryGetValue("X-GitHub-Delivery", out StringValues gitHubDeliveryId);
        Request.Headers.TryGetValue("X-GitHub-Event", out StringValues gitHubEvent);
        Request.Headers.TryGetValue("X-Hub-Signature", out StringValues gitHubSignature);

        _logger.LogInformation("Received GitHub delivery {GitHubDeliveryId} for event {gitHubEvent}", gitHubDeliveryId, gitHubEvent);

        using (var reader = new StreamReader(Request.Body))
        {
            var txt = await reader.ReadToEndAsync();

            if (IsGitHubSignatureValid(txt, gitHubSignature))
            {
                return Ok("works with configured secret!");
            }
        }

        return Unauthorized();
    }
}
```

## Deploying the application

With the webhook done, we can now deploy it to AWS. Right-click on the project and select the **Publish to AWS Lambda...** option:

![Select publish to Lambda menu item](/assets/images/2018-01-23-create-github-webhook-aspnetcore-aws-lambda/publish-to-lambda-menu.png)

In the **Publish AWS Serverless Application** dialog, create a new Cloudformation stack and S3 bucket to be used by CloudFormation. Once done you can click the **Publish** button:

![Publish AWS Serverless application](/assets/images/2018-01-23-create-github-webhook-aspnetcore-aws-lambda/publish-aws-serverless-application.png)

The AWS Toolkit will now take over and deploy the application using the serverless template and CloudFormation. Once the process has completed, you can copy the value of the **AWS Serverless URL**, as we will need that next to configure the webhook in GitHub:

![CloudFormation Stack is published](/assets/images/2018-01-23-create-github-webhook-aspnetcore-aws-lambda/cloudformation-stack-published.png)

Before doing that, however, we will need to configure the value of the **GitHubWebhookSecret** environment variable. In the AWS Console, locate the Lambda function which was created for you by the CloudFormation serverless template. Scroll down to the **Environment variables** section and add a **GitHubWebhookSecret** variable with whatever **secret** value you decide.

Make sure to **Save** the changes to the Lambda function when done.

![Ad the environment variable for the webhook secret](/assets/images/2018-01-23-create-github-webhook-aspnetcore-aws-lambda/add-webhook-secret-environment-variable.png)

> For heaven's sake, please do not use the value _"mysecret"_ like I did in the sample screenshot above. Use a password manager like 1Password to create a proper secret value...

## Configuring the webhook in GitHub

Go to the GitHub repository for which you want to configure a webhook, and in the **Webhook** tab of the **Settings**, click on the **Add webhook** button:

![Add a new webhook](/assets/images/2018-01-23-create-github-webhook-aspnetcore-aws-lambda/add-new-webhook.png)

Specify the **Payload URL** (for my webhook this was `https://o8opr7cuxa.execute-api.us-east-2.amazonaws.com/Prod/webhooks/github`) and the same **secret** you configured for your Lambda function. You can also specify which events you are interested in. In my case, I opted to receive all events.

Once done you can click on the **Add webhook** button.

![Specify details for new webhook](/assets/images/2018-01-23-create-github-webhook-aspnetcore-aws-lambda/webhook-details.png)

## Testing it out

At this point, GitHub would have already sent a ping event to ensure that the webhook is configured correctly. In the AWS Console, you can go to the _Monitoring_ tab for your Lambda function where you will see the CloudWatch metrics and will notice that an invocation was already made.

Click on the **Jump to Logs** link to view the CloudWatch logs:

![View Lambda function Monitoring section](/assets/images/2018-01-23-create-github-webhook-aspnetcore-aws-lambda/webhook-monitoring.png)

You will notice the ping event receive from GitHub

![View the CloudWatch Logs for the Lambda function](/assets/images/2018-01-23-create-github-webhook-aspnetcore-aws-lambda/cloudwatch-logs-1.png)

Let's head back to GitHub and create a new issue on the repository:

![Create a new GitHub issue](/assets/images/2018-01-23-create-github-webhook-aspnetcore-aws-lambda/create-github-issue.png)

Once the issue has been created, you and head back to CloudWatch and refresh your logs. Sure enough, there is the new event:

![View the CloudWatch Logs for the Lambda function](/assets/images/2018-01-23-create-github-webhook-aspnetcore-aws-lambda/cloudwatch-logs-2.png)

> You may have to wait a few seconds for GitHub to send the WebHook, Lambda to process it, and the CloudWatch logs to be updated.

## Conclusion

In this blog post, I demonstrated how to create a GitHub webhook using ASP.NET Core, AWS API Gateway and AWS Lambda. Even though the ASP.NET Webhooks libraries are not yet available for ASP.NET Core, it is still a pretty straightforward process.

Source code is available at [https://github.com/jerriepelser-blog/LambdaGitHubWebhook](https://github.com/jerriepelser-blog/LambdaGitHubWebhook)