---
date: 2017-04-11T00:00:00Z
description: |
  A look at creating a simple serverless application using .NET Core, AWS Lambda and AWS API Gateway
tags:
- .net core
- asp.net
- asp.net core
- aws
- aws lambda
- aws api gateway
title: Creating a Serverless Application with .NET Core, AWS Lambda and AWS API Gateway
url: /blog/dotnet-core-aws-lambda-serverless-application/
---

Previously [I gave an overview](/blog/getting-started-with-dotnet-core-aws-lambda/) of the programming models when using NET Core with AWS Lambda, and I also showed how to create an [image compressor in Lambda and C#](/blog/compress-images-csharp-lambda-tinypng/).

This time around we'll put together a simple Web API with a couple of endpoints which can be called from any client application. The API I'll create will utilize [NodaTime](http://nodatime.org/) library created by Jon Skeet to return a list of time zones based on the [Time Zone database](https://www.iana.org/time-zones).

There will be 2 endpoints:

1. The first endpoint will be at `/zones` and will return a list of all time zones.
2. The second endpoint will return only a single time zone, and the endpoint will accept a request in the format `/zones/{id}` where `id` is the ID of the time zone to return.

## Create the project

As before, ensure you have downloaded and installed the [Preview of the AWS Toolkit for Visual Studio 2017](https://aws.amazon.com/blogs/developer/preview-of-the-aws-toolkit-for-visual-studio-2017/).

Inside Visual Studio, create a new project and select the **AWS Serverless Application (.NET Core)** template:

![](/assets/images/2017-04-11-dotnet-core-aws-lambda-serverless-application/new-project.png)

Next select the **Empty Serverless Application** blueprint

![](/assets/images/2017-04-11-dotnet-core-aws-lambda-serverless-application/select-blueprint.png)

Ensure that you have Nodatime installed using NuGet:

```text
Install-Package NodaTime
```

## Return the list of time zones

First up, open the `serverless.template` file, and alter the default file that was created by the template. Change the name of the resource to **GetAll**, the **Handler** to point to a function called **GetAllTimeZones** and set the **Path** to **/zones**: 

```json
{
  "AWSTemplateFormatVersion" : "2010-09-09",
  "Transform" : "AWS::Serverless-2016-10-31",
  "Description" : "An AWS Serverless Application.",

  "Resources" : {

    "GetAll" : {
      "Type" : "AWS::Serverless::Function",
      "Properties": {
        "Handler": "TimeZoneService::TimeZoneService.Functions::GetAllTimeZones",
        "Runtime": "dotnetcore1.0",
        "CodeUri": "",
        "MemorySize": 256,
        "Timeout": 30,
        "Role": null,
        "Policies": [ "AWSLambdaBasicExecutionRole" ],
        "Events": {
          "PutResource": {
            "Type": "Api",
            "Properties": {
              "Path": "/zones",
              "Method": "GET"
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

This will create a Lambda function called **GetAll** and will register the `/zones` endpoint in AWS API Gateway to call this function. Inside our application, the function called `GetAllTimeZones` will handle this request.

Please note that in the `serverless.template` file the full value for **Handler** is set to **TimeZoneService::TimeZoneService.Functions::GetAllTimeZones**. This will call a function in the format **assembly::namespace.class::method**. So in other words the handler for the function is located in the assembly **TimeZoneService**, namespace **TimeZoneService** and class **Functions**. The method inside that class which will handle the request is the function **GetAllTimeZones**.

So let's create this function. Head over to the `Functions.cs` class which was created by the template. You can delete the default function which was created by the template and replace the class with the following code:

```csharp
namespace TimeZoneService
{
    public class Functions
    {
        public APIGatewayProxyResponse GetAllTimeZones(APIGatewayProxyRequest request, ILambdaContext context)
        {
            List<TimeZoneInfo> timeZones = new List<TimeZoneInfo>();

            foreach (var location in TzdbDateTimeZoneSource.Default.ZoneLocations)
            {
                timeZones.Add(GetZoneInfo(location));
            }

            var response = new APIGatewayProxyResponse
            {
                StatusCode = (int)HttpStatusCode.OK,
                Body = JsonConvert.SerializeObject(timeZones),
                Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
            };

            return response;
        }

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
    }
}
```

The `GetAllTimeZones` function will simply iterate through all the locations in the [TZDB](https://www.iana.org/time-zones) database. For each location it will do some calculation to determine the details of the time zone for that location, such as the offset from UTC, as well as the offset during Daylight Savings, if applicable.

Next up we can deploy the project. Right click on the project in the Solution Explorer, and select the **Publish to AWS Lambda...** option. The **Publish AWS Serverless Application** dialog will be displayed. Complete the required information, and deploy the project.

![](/assets/images/2017-04-11-dotnet-core-aws-lambda-serverless-application/publish-aws-serverless-application.png)

Once the project has been deployed, the URL for your serverless app will be displayed in the **AWS Serverless URL** field:

![](/assets/images/2017-04-11-dotnet-core-aws-lambda-serverless-application/application-published.png)

Go to that URL and append `/zones` to access the endpoint to retrieve all the time zones:

![](/assets/images/2017-04-11-dotnet-core-aws-lambda-serverless-application/postman-all-zones.png)

## Return a single time zone

Next up, let's create a method that will return an single time zone. For this we can create a new method called `GetSingleTimeZone`:

```csharp
public APIGatewayProxyResponse GetSingleTimeZone(APIGatewayProxyRequest request, ILambdaContext context)
{
    string timeZoneId = null;
    if (request.PathParameters != null && request.PathParameters.ContainsKey("Id"))
        timeZoneId = request.PathParameters["Id"];

    if (!String.IsNullOrEmpty(timeZoneId))
    {
        // Url decode the TZID
        timeZoneId = WebUtility.UrlDecode(timeZoneId);

        var location = TzdbDateTimeZoneSource.Default.ZoneLocations.FirstOrDefault(
            l => String.Compare(l.ZoneId, timeZoneId, StringComparison.OrdinalIgnoreCase) == 0);

        if (location != null)
        {
            return new APIGatewayProxyResponse
            {
                StatusCode = (int)HttpStatusCode.OK,
                Body = JsonConvert.SerializeObject(GetZoneInfo(location)),
                Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
            };
        }
    }
           
    return new APIGatewayProxyResponse
    {
        StatusCode = (int)HttpStatusCode.NotFound
    };
}
```

This method will check to see if there is a path parameter called `Id` which will contain the value for the time zone ID you want to retrieve. If the parameter is present it will be retrieved and URL-decoded. Then we'll retrieve the time zone with that ID, and as before we'll retrieve the information for the time zone.

Finally, if the `Id` parameter was not passed, or a time zone location with that ID was not found, then an HTTP Status 404 will be returned.

Also be sure to update your `serverless.template` file and add the settings for the new endpoint. Add a resource called **GetSingle** which will call the **GetSingleTimeZone** method. For the **path** you can specify `/zones/{Id}`. This will ensure that the path parameter called `Id` will be passed through to the method, which you remember we retrieved from the `PathParameters` collection.

**NB:** The name of the parameter is case-sensitive so be sure that if you specify the parameter name as `Id` in the `serverless.template` file that you check for exactly that name in your C# code.

```json
{
  "AWSTemplateFormatVersion" : "2010-09-09",
  "Transform" : "AWS::Serverless-2016-10-31",
  "Description" : "An AWS Serverless Application.",

  "Resources" : {

    "GetAll" : {
      ...
    },
	"GetSingle" : {
      "Type" : "AWS::Serverless::Function",
      "Properties": {
        "Handler": "TimeZoneService::TimeZoneService.Functions::GetSingleTimeZone",
        "Runtime": "dotnetcore1.0",
        "CodeUri": "",
        "MemorySize": 256,
        "Timeout": 30,
        "Role": null,
        "Policies": [ "AWSLambdaBasicExecutionRole" ],
        "Events": {
          "PutResource": {
            "Type": "Api",
            "Properties": {
              "Path": "/zones/{Id}",
              "Method": "GET"
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

Deploy the application as before, and make a request to the `/zones` endpoint, this time passing the ID for the single time zone you want to retrieve. Note that the time zone ID can contain a slash (`/`), so it needs to be URL encoded otherwise it will be interpreted as a path separator.

![](/assets/images/2017-04-11-dotnet-core-aws-lambda-serverless-application/postman-single-zone.png)

## Source Code 

The source code is available at [https://github.com/jerriepelser-blog/aws-dotnet-serverless-app](https://github.com/jerriepelser-blog/aws-dotnet-serverless-app)

