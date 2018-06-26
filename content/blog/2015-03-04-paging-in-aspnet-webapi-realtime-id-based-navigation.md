---
date: 2015-03-04T00:00:00Z
description: |
  Looking at how we can page over real-time data by using the ID of the record
tags:
- aspnet
- aspnet mvc
- aspnet web api
title: 'Paging in ASP.NET Web API: Real-time ID-based Navigation'
url: /blog/paging-in-aspnet-webapi-realtime-id-based-navigation/
---

## Introduction

The previous three posts in this series have focused on paging over a relatively static data set, such as a customer list. When working with real-time data such as a Twitter feed, where new records are constantly being added, you can run into a situation where records are either duplicated between pages, or some records are completely skipped over. Rakhitha Nimesh has written an article about this entitled [Paginating Real-Time Data with Cursor Based Pagination](http://www.sitepoint.com/paginating-real-time-data-cursor-based-pagination/). I am not going to try and reproduce his well written explanation, so I suggest you go and read that quickly to understand the underlying problems.

Just to be clear, the problems you encounter when paging over real-time data is certainly also applicable to even fairly static sets of data such as a customer list. It is just that it is far more likely to occur in a real-time data scenario.

## Setting the table

For this blog post I will create a fictional `Tweet` class:

``` csharp
public class Tweet
{
    public int Id { get; set; }

    public DateTime CreatedDate { get; set; }

    [Required]
    public string Text { get; set; }
}
```

And also update the database context to add the list of tweets:

``` csharp
public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
{
    public IDbSet<Customer> Customers { get; set; }

    public IDbSet<Tweet> Tweets { get; set; }

    public ApplicationDbContext()
        : base("DefaultConnection", throwIfV1Schema: false)
    {
    }
    
    public static ApplicationDbContext Create()
    {
        return new ApplicationDbContext();
    }
```

After that create a database migration, and once again I use the combination of [NBuilder](http://www.nbuilder.org) and [Faker.NET](https://www.nuget.org/packages/Faker.Net) to generate some fake Lorem Ipsum tweets:

``` csharp
internal sealed class Configuration : DbMigrationsConfiguration<WebApiPaging.Models.ApplicationDbContext>
{
    public Configuration()
    {
        AutomaticMigrationsEnabled = false;
    }

    protected override void Seed(WebApiPaging.Models.ApplicationDbContext context)
    {
		...

        // If no tweets exist, add them
        if (!context.Tweets.Any())
        {
            var createdDate = DateTime.Now.AddDays(-2);

            var tweets = Builder<Tweet>.CreateListOfSize(1000)
                .All()
                .With(tweet =>
                {
                    createdDate = createdDate.AddSeconds(1);
                    tweet.CreatedDate = createdDate;

                    return tweet;
                })
                .With(tweet => tweet.Text = Faker.Lorem.Sentence())
                .Build()
                .ToArray();

            context.Tweets.AddOrUpdate(t => t.Id, tweets);
        }
    }
}
```

## Defining the API

Before we get into the coding I want to have a quick overview of the API methods I will be building and what they will be returning. First of all remember that the tweets all have a sequential ID as primary key - in my example this is a simple auto increment field. Tweets with a lower ID will therefore be older that tweets with a higher ID - i.e. the tweet with ID 1 was added before the tweet with ID 1000. When requesting tweets the user of the API can specify a `count` parameter to limit the number of tweets which are returned - by default we will limit this to 50 tweets. 

To navigate through all the tweets the user will use either a `before` or `after` parameter. When specifying the `before` parameter we will return tweets with an ID lower than that. So specifying the `before` value of 400 for example, we will then return the tweets with IDs lower that that (e.g. 399, 398, 397 etc.). When specifying an `after` parameter we will return tweets with IDs higher that that. So if the user specifies the `after` parameter with the value of 400, we will return tweets with IDs 401, 401, 402 etc.

The result from the API call will always just be a simple JSON array containing the tweets. There will be no extra paging data returned because it really does not make sense, since there is no concept of a page or page numbers. Also since we assume this is real-time data, returning the number of records also makes little sense because it will probably be outdated by the time the user gets it back from the API.

To "page", the user can look at the lowest and highest IDs in the result and then use that to request the next batch of tweets using the `before` and `after` parameters. Also, when no `before` or `after` parameters are specified, I will return the latest `count` number of records.

Just to visualize this quickly, let us assume the user makes the following request:

``` text
GET /tweets
```

No count parameter is specified so I will assume 50 records, and no before or after is specified so I assume the latest 50 records. So I will return the following for example:

``` json
[
	{
		"id": 1000,
		"text": "some witty text"
	},
	{
		"id": 999,
		"text": "..."
	},
	{
		"id": 998,
		"text": "..."
	},
	...
	{
		"id": 951,
		"text": "..."
	}
]
```

Should the user want to get older tweets, they then look at the result and see that the lowest ID is 951, so they make the following request:

``` text
GET /tweets?before=951
```

Once again no `count` is specified so I assume 50 records. This time the `before` parameter is specifed, so I will return the first 50 records with IDs lower than 951. For example:

``` json
[
	{
		"id": 950,
		"text": "..."
	},
	{
		"id": 949,
		"text": "..."
	},
	{
		"id": 948,
		"text": "..."
	},
	...
	{
		"id": 901,
		"text": "..."
	}
]
```

And similarly to get all newer tweets, they could have made the following request:

``` text
GET /tweets?after=1000
``` 

Obviously it would not make sense to specify both a `before` and `after` parameter, and how you handle this is your choice. You can return an error to the user, but what I will do in this post is to assume that `after` takes precedence, so in the instance where both parameters are specied I will assume that the user wants newer records and only use the value of the `after` parameter.

## Let's get coding...

With all that, the actual code is fairly simple. Here is the full code for the new controller:

``` csharp
[RoutePrefix("api")]
public class TweetsController : ApiController
{
    private readonly ApplicationDbContext _dbContext;

    public TweetsController()
    {
        _dbContext = new ApplicationDbContext();   
    }

    [HttpGet]
    [Route("tweets")]
    public IHttpActionResult GetTweets(int count = 50, int? after = null, int? before = null)
    {
        List<Tweet> tweets;

        if (after.HasValue)
        {
            // if the user specified the after parameter, return the next batch of records with a higher ID
            tweets = _dbContext.Tweets
                .Where(t => t.Id > after)
                .OrderBy(t => t.Id)
                .Take(count)
                .ToList();
        }
        else if (before.HasValue)
        {
            // if the user specified the before parameter, return the next batch of records with a lower ID
            tweets = _dbContext.Tweets
                .Where(t => t.Id < before)
                .OrderByDescending(t => t.Id)
                .Take(count)
                .ToList();
        }
        else
        {
            // If neither is specified return the latest records
            tweets = _dbContext.Tweets
                .OrderByDescending(t => t.Id)
                .Take(count)
                .ToList();
        }

        return Ok(tweets.OrderBy(t => t.Id));
    }
}
``` 

As you can see, when the user specifies the `after` parameter, I simply look for tweets with IDs greater than that. I also specify and `OrderBy` to ensure that only the next `count` number of records immediately following that ID is selected. Similarly, when the user specifies the `before` parameter I select the tweets with an ID lower that that. In this instance I order the records in descending order to ensure get the next sequential `count` number of records immediately lower that the ID specified. 

When neither a `before` or `after` parameter is specified I simply order the tweets in descending order to have the latest ones first and then select `count` number of records.

The final thing I do when returning the tweets is to order them in ascending order to ensure that the tweet with the lowest ID always appears first in the JSON array which I return. But this is an implementation detail of how I choose to do it. Since it is real-time data you would probably want to have the latest tweet appear first in the JSON array. 

I would however suggest that you do sort the elements before you return them to make sure that they are always returned in the same order, and not sometimes in ascending and sometimes in descending order (depending on whether the user specified a `before` or `after` parameter).

So with that let us head over to [Postman](http://www.getpostman.com/) again and make the following request:

``` text
http://localhost:7965/api/tweets
```

The result as you can see is the latest 50 tweets, i.e from ID 951 through ID 1000:

![](/assets/images/2015-03-04-paging-in-aspnet-webapi-realtime-id-based-navigation/postman-1.png)

And then to select the next batch of tweets before that, request all the records `before` ID 951:

``` text
http://localhost:7965/api/tweets?before=951
```

And the result is tweets with IDs 901 through 950:

![](/assets/images/2015-03-04-paging-in-aspnet-webapi-realtime-id-based-navigation/postman-2.png)

## Conclusion

In this blog post I demonstrated how you can page through sets of real-time data. Since the data is constantly in flux, the traditional paged-based mechanism in the previous blog posts are not applicable (or rather not very useful). I demonstrated a fictitional set of tweets which has a sequential integer-based primary key where each next tweet is assigned a higher ID value. You can then select pages (or batches) or tweets based on ID and selecting tweets with either a lower ID (i.e. older) or a higher ID (i.e. newer). 

You do not have to use an auto increment ID, but can also use any other form of sequential ID such as a timestamp.

This is the final post in this series, and I hoped you enjoyed it and that you got value from it.