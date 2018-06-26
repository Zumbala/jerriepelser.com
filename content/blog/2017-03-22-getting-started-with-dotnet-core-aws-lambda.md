---
date: 2017-03-22T00:00:00Z
description: |
  A look at some resources to get you started with developing .NET Core applications hosted on AWS Lambda
tags:
- .net core
- asp.net
- asp.net core
- aws
- aws lambda
- aws api gateway
title: Getting started with .NET Core and AWS Lambda
url: /blog/getting-started-with-dotnet-core-aws-lambda/
---

This blog post will provide you with a brief introduction to using C# and .NET Core with [AWS Lambda](https://aws.amazon.com/lambda/) and also look at the different programming models available when using [.NET Core](https://www.microsoft.com/net/core/platform) with Lambda.

## Serverless and AWS Lambda 

AWS Lambda is the [serverless](https://auth0.com/blog/what-is-serverless/) product offered by Amazon Web Services. Serverless does not mean that there is no server involved - obviously there is - but just that managing servers, scaling, etc is not something that you need to worry about. 

What you need to worry about (as programmer) is writing the code that executes the logic. You can then deploy it to a serverless environment such as AWS Lambda and it will take care of scaling etc. Some of the other offerings available are [Azure Functions](https://azure.microsoft.com/en-us/services/functions/), [Google Cloud Functions](https://cloud.google.com/functions/) and also [Webtask](https://webtask.io/).

The other advantage is that serverless environments only charge you for the time your code takes to execute. So if you have a small POC you want to develop, this is ideal as you do not have to worry about paying for a server which sits idle 99% of the time.

## Programming models for C# / .NET Core

I have only started playing around with AWS Lambda recently, but from what I can see so far there are basically 3 models which you can use when developing AWS Lambda functions using .NET Core and C#.

* **Plain Lambda Function.** In this case you create a Lambda function which is a simple C# class with a function. This function can be triggered by [certain events](http://docs.aws.amazon.com/lambda/latest/dg/invoking-lambda-function.html), or triggered manually from an application.
* **Serverless Application.** In this case you can deploy an AWS Lambda function (or collection of functions) using [AWS CloudFormation](https://aws.amazon.com/cloudformation/) and front it with [AWS API Gateway](https://aws.amazon.com/api-gateway/). 

    You can also read more about this model in [Deploying Lambda-based Applications](http://docs.aws.amazon.com/lambda/latest/dg/deploying-lambda-apps.html).
* **ASP.NET Core app as Serverless Application.** This is a variant of the model above, but in this instance your entire exiting ASP.NET Core application is published as a single Lambda function. This then utilizes the API Gateway Proxy Integration to forward all request coming through API Gateway to this Lambda Function.

    The Lambda function then hands the request off to the ASP.NET Core pipeline. So you have all the middleware you love at your disposal. 
    
    This means that you basically have the normal ASP.NET Core request pipeline, but instead of IIS/Nginx and Kestrel, you have API Gateway and Lambda. So instead of this:

    ![](/assets/images/2017-03-22-getting-started-with-dotnet-core-aws-lambda/normal-flow.png)

    You have this:

    ![](/assets/images/2017-03-22-getting-started-with-dotnet-core-aws-lambda/serverless-flow.png)

In the coming blog posts I will dive deeper into each of these.

## So why AWS Lambda and not any of the others?

So I guess another question is why I am using AWS Lambda and not the offerings from Microsoft, Google or Auth0's own WebTask? Well, for a couple of reasons **which are true at the time of writing this blog post**:

1. It supports .NET Core - the others don't. Google Functions isn't even generally available yet.
2. The 3rd programming model I mentioned above (ASP.NET Core app as Serverless Application) is one which interests me the most. I can take an entire ASP.NET Core app and publish it to Lambda. This means I can test my app locally with my normal development flow, and then simply push it to AWS to host it as a Lambda function.

## Some resources to get you started

* [Serverless Development with C# and AWS Lambda (video from re:Invent 2016)](https://www.youtube.com/watch?v=Ymn6WGCSjE4). 

    I recommend watching this video first as it gives a great overview of AWS Lambda, the C# and Visual Studio support for AWS Lambda as well as the 3 programming models I discussed above.
    
* [Preview of the AWS Toolkit for Visual Studio 2017](https://aws.amazon.com/blogs/developer/preview-of-the-aws-toolkit-for-visual-studio-2017/)
* [Using the AWS Lambda Project in Visual Studio](https://aws.amazon.com/blogs/developer/using-the-aws-lambda-project-in-visual-studio/)
* [AWS Serverless Applications in Visual Studio](https://aws.amazon.com/blogs/developer/aws-serverless-applications-in-visual-studio/)
* [Running Serverless ASP.NET Core Web APIs with Amazon Lambda](https://aws.amazon.com/blogs/developer/running-serverless-asp-net-core-web-apis-with-amazon-lambda/)
* [Deploy an Existing ASP.NET Core Web API to AWS Lambda](https://aws.amazon.com/blogs/developer/category/net/)
* [Deploying .NET Core AWS Lambda Functions from the Command Line](https://aws.amazon.com/blogs/developer/deploying-net-core-aws-lambda-functions-from-the-command-line/)
* [Creating .NET Core AWS Lambda Projects without Visual Studio](https://aws.amazon.com/blogs/developer/creating-net-core-aws-lambda-projects-without-visual-studio/)

Also stay tuned, as I plan to do more posts on C#, .NET Core and Lambda in the future.