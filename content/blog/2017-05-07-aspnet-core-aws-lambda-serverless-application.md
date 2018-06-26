---
date: 2017-05-07T00:00:00Z
description: |
  Creating a simple serverless application using ASP.NET Core, AWS Lambda and AWS API Gateway
tags:
- .net core
- asp.net
- asp.net core
- aws
- aws lambda
- aws api gateway
title: Creating a Serverless Application with ASP.NET Core, AWS Lambda and AWS API
  Gateway
url: /blog/aspnet-core-aws-lambda-serverless-application/
---

In my [previous blog post](/blog/dotnet-core-aws-lambda-serverless-application/) I gave an overview of how you can create a serverless application using .NET Core and [AWS Lambda](https://aws.amazon.com/lambda/). 

In this blog post I will show a similar - yet quite different - programming model. Instead of using just plain .NET Core, I'll be using ASP.NET Core with AWS Lambda. A quick reminder from my [introductory blog post in this series](/blog/getting-started-with-dotnet-core-aws-lambda/) may be in order.

Remember that for a typical ASP.NET Core application, a request to your website will go through IIS/Nginx, then Kestrel, and finally will be passed on to ASP.NET Core:

![](/assets/images/2017-05-07-aspnet-core-aws-lambda-serverless-application/normal-flow.png)

In the case of an ASP.NET Core Serverless application however, this is a bit different. The request will come in through API Gateway, which will pass it on to Lambda, which in turn will pass it on to ASP.NET Core:

![](/assets/images/2017-05-07-aspnet-core-aws-lambda-serverless-application/serverless-flow.png)

## Overview of the application

As with the previous blog post, we'll put together a simple Web API with a couple of endpoints which can be called from any client application. The API I'll create will utilize [NodaTime](http://nodatime.org/) library created by Jon Skeet to return a list of time zones based on the [Time Zone database](https://www.iana.org/time-zones).

There will be 2 endpoints:

1. The first endpoint will be at `/zones` and will return a list of all time zones.
2. The second endpoint will return only a single time zone, and the endpoint will accept a request in the format `/zones/{id}` where `id` is the ID of the time zone to return.

## Create the project

Before creating the application, ensure you have downloaded and installed the [Preview of the AWS Toolkit for Visual Studio 2017](https://aws.amazon.com/blogs/developer/preview-of-the-aws-toolkit-for-visual-studio-2017/).

Inside Visual Studio, create a new project and select the **AWS Serverless Application (.NET Core)** template:

![](/assets/images/2017-05-07-aspnet-core-aws-lambda-serverless-application/new-project-dialog.png)

Next select the **ASP.NET Core Web API** blueprint:

![](/assets/images/2017-05-07-aspnet-core-aws-lambda-serverless-application/new-aws-serverless-application.png)

Ensure that you have Nodatime installed using NuGet:

```text
Install-Package NodaTime
```

## Some cleanup first

The project created from the template will contain a lot of extra stuff which is not needed in for this application, most notably `ValuesController` and `S3ProxyController` classes in the `Controllers` folder. Go ahead and delete the files for those classes.

It will also add some unneeded entries in the `serverless.template` file. Get rid of all of the entries related to S3, after which you should be left with a file that looks like the following:

```json
{
  "AWSTemplateFormatVersion" : "2010-09-09",
  "Transform" : "AWS::Serverless-2016-10-31",
  "Description" : "An AWS Serverless Application that uses the ASP.NET Core framework running in Amazon Lambda.",
  "Parameters" : {
  },
  "Conditions" : {
  },
  "Resources" : {
    "Get" : {
      "Type" : "AWS::Serverless::Function",
      "Properties": {
        "Handler": "TimeZoneService::TimeZoneService.LambdaEntryPoint::FunctionHandlerAsync",
        "Runtime": "dotnetcore1.0",
        "CodeUri": "",
        "MemorySize": 256,
        "Timeout": 60,
        "Role": null,
        "Policies": [ "AWSLambdaFullAccess" ],
        "Environment" : {
          "Variables" : {
          }
        },
        "Events": {
          "PutResource": {
            "Type": "Api",
            "Properties": {
              "Path": "/{proxy+}",
              "Method": "ANY"
            }
          }
        }
      }
    }
  },
  "Outputs" : {
  }
}
```

## Return the list of time zones

Create a new MVC Controller called `ZonesController` and decorate it with the attribute `[Route("[controller]")]`. This will ensure that the controller actions will be available at the route with the same name as the controllers - in this case `/zones`.

Create a class called `TimeZoneInfo`. This is the class which will contain the information for a particular time zone.

```csharp
public class TimeZoneInfo
{
    public string CountryCode { get; set; }
    public string CountryName { get; set; }
    public TimeSpan DstOffset { get; set; }
    public TimeSpan Offset { get; set; }
    public string TimeZoneId { get; set; }
}
```

And also a helper function called `GetZoneInfo`, which will get the information of a particular time zone, as well as do some calculations to determine whether daylight savings is applicable during this current year for the time zone to be able to return the appropriate information:

```csharp
private TimeZoneInfo GetZoneInfo(TzdbZoneLocation location)
{
    var zone = DateTimeZoneProviders.Tzdb[location.ZoneId];

    // Get the start and end of the year in this zone
    var startOfYear = zone.AtStartOfDay(new LocalDate(2017, 1, 1));
    var endOfYear = zone.AtStrictly(new LocalDate(2018, 1, 1).AtMidnight().PlusNanoseconds(-1));

    // Get all intervals for current year
    var intervals = zone.GetZoneIntervals(startOfYear.ToInstant(), endOfYear.ToInstant()).ToList();

    // Try grab interval with DST. If none present, grab first one we can find
    var interval = intervals.FirstOrDefault(i => i.Savings.Seconds > 0) ?? intervals.FirstOrDefault();

    return new TimeZoneInfo
    {
        TimeZoneId = location.ZoneId,
        Offset = interval.StandardOffset.ToTimeSpan(),
        DstOffset = interval.WallOffset.ToTimeSpan(),
        CountryCode = location.CountryCode,
        CountryName = location.CountryName
    };
}
```

Now all that is left for us to do is to write the controller action which will iterate through the list of time zones, call the `GetZoneInfo` method for each, and finally return a list of all time zones:

```csharp
[Route("[controller]")]
public class ZonesController : Controller
{
    [HttpGet]
    public IActionResult Get()
    {
        List<TimeZoneInfo> timeZones = new List<TimeZoneInfo>();

        foreach (var location in TzdbDateTimeZoneSource.Default.ZoneLocations)
        {
            timeZones.Add(GetZoneInfo(location));
        }

        return Ok(timeZones);
    }

    private TimeZoneInfo GetZoneInfo(TzdbZoneLocation location)
    {
        // Code omitted for brevity
    }
}

```

## Local and Lambda entry points

Testing an ASP.NET Core Lambda Serverless application is a major plus point for me over the normal .NET Lambda Serverless application. With the normal application you actually have to deploy the application to AWS to be able to test it. With and ASP.NET Core Lambda Serverless application however, you can run the application locally as you would with any normal ASP.NET Core application.

The reason for this is because the ASP.NET Core serverless application contains a class called `LocalEntryPoint` which does the same bootstrapping of the hosting environment [as a normal ASP.NET Core application would do](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/hosting):

```csharp
public class LocalEntryPoint
{
    public static void Main(string[] args)
    {
        var host = new WebHostBuilder()
            .UseKestrel()
            .UseContentRoot(Directory.GetCurrentDirectory())
            .UseIISIntegration()
            .UseStartup<Startup>()
            .Build();

        host.Run();
    }
}
```

However it also contains a second entry point called `LambdaEntryPoint`:

```csharp
public class LambdaEntryPoint : Amazon.Lambda.AspNetCoreServer.APIGatewayProxyFunction
{
    protected override void Init(IWebHostBuilder builder)
    {
        builder
            .UseContentRoot(Directory.GetCurrentDirectory())
            .UseStartup<Startup>()
            .UseApiGateway();
    }
}
```

This class is configured inside the `serverless.template` as the handler for all traffic coming through API Gateway. Let's have a look at that `serverless.template` file again:

```json
{
  "AWSTemplateFormatVersion" : "2010-09-09",
  "Transform" : "AWS::Serverless-2016-10-31",
  "Description" : "An AWS Serverless Application that uses the ASP.NET Core framework running in Amazon Lambda.",
  "Parameters" : {
  },
  "Conditions" : {
  },
  "Resources" : {
    "Get" : {
      "Type" : "AWS::Serverless::Function",
      "Properties": {
        "Handler": "TimeZoneService::TimeZoneService.LambdaEntryPoint::FunctionHandlerAsync",
        "Runtime": "dotnetcore1.0",
        "CodeUri": "",
        "MemorySize": 256,
        "Timeout": 60,
        "Role": null,
        "Policies": [ "AWSLambdaFullAccess" ],
        "Environment" : {
          "Variables" : {
          }
        },
        "Events": {
          "PutResource": {
            "Type": "Api",
            "Properties": {
              "Path": "/{proxy+}",
              "Method": "ANY"
            }
          }
        }
      }
    }
  },
  "Outputs" : {
  }
}
```

See that `{proxy+}` in the path? This means that this function handler is configured as a [Proxy Resource](http://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-set-up-simple-proxy.html), and all traffic coming in on that path will be sent on to the `FunctionHandlerAsync` method. This particular method exists on the `APIGatewayProxyFunction` base class, and is part of the [AWS Lambda for .NET Core](https://github.com/aws/aws-lambda-dotnet) library. 

You can go and have a look at [the source code for that function](https://github.com/aws/aws-lambda-dotnet/blob/f1eb9b067b5b6a5a5dde9284b42762ec4606390f/Libraries/src/Amazon.Lambda.AspNetCoreServer/APIGatewayProxyFunction.cs#L117-L131), but what it essentially does is to marshall the request coming in from API Gateway to the ASP.NET Core pipeline. From that point the normal ASP.NET Core pipeline takes over and you have all the greatness that comes with that: routing, model binding, etc...

Oh and did I mention that you can test and debug locally? So let's do that by running the application...

## Testing the application

Once the application is running you can open [Postman](https://www.getpostman.com/) and make a request to `http://localhost:5000/zones` and you will see the list of time zones returned:

![](/assets/images/2017-05-07-aspnet-core-aws-lambda-serverless-application/postman-all-zones.png)

## Return a single time zone

Next up, let's create a controller action that will return an single time zone. For this method we will be passed in a route parameter called `id` which will contain the time zone ID you want to retrieve. This parameter will be something like `Europe/Andorra`, so because it contains the path character (`/`), it will actually be passed in as a URL encoded string, e.g. `Europe%2FAndorra`.

Under normal circumstances the ASP.NET MVC [model binding](https://docs.microsoft.com/en-us/aspnet/core/mvc/models/model-binding) will take care of the URL decoding, but it seems that for the path character (`/`) it [will not do this](https://github.com/aspnet/Mvc/issues/4599), so we need to take care of that and do the URL decoding manually.

Here is the code for the action:

```csharp
[HttpGet("{id}")]
public IActionResult Get([FromRoute] string id)
{
    if (!String.IsNullOrEmpty(id))
    {
        string timeZoneId = WebUtility.UrlDecode(id);

        var location = TzdbDateTimeZoneSource.Default.ZoneLocations.FirstOrDefault(
            l => String.Compare(l.ZoneId, timeZoneId, StringComparison.OrdinalIgnoreCase) == 0);

        if (location != null)
        {
            return Ok(GetZoneInfo(location));
        }
    }

    return NotFound();
}
```

Once again, let's fire up the application and hit that endpoint in Postman:

![](/assets/images/2017-05-07-aspnet-core-aws-lambda-serverless-application/postman-single-zone.png)

## Deploying to AWS

The last bit I want to demonstrate is how to deploy the project to Lambda. 

Right click on the project in the Solution Explorer, and select the **Publish to AWS Lambda...** option:

![](/assets/images/2017-05-07-aspnet-core-aws-lambda-serverless-application/deploy-menu.png)

The **Publish AWS Serverless Application** dialog will be displayed. Complete the required information, and deploy the project.

![](/assets/images/2017-05-07-aspnet-core-aws-lambda-serverless-application/publish-dialog.png)

Once the project has been deployed, the URL for your serverless app will be displayed in the **AWS Serverless URL** field:

![](/assets/images/2017-05-07-aspnet-core-aws-lambda-serverless-application/aws-deployed.png)

Once again we can use Postman to test our API. Go to that URL and append `/zones` to access the endpoint to retrieve all the time zones:

![](/assets/images/2017-05-07-aspnet-core-aws-lambda-serverless-application/postman-all-zones-aws.png)

## Source Code 

The source code is available at [https://github.com/jerriepelser-blog/aws-aspnet-serverless-app](https://github.com/jerriepelser-blog/aws-aspnet-serverless-app)
