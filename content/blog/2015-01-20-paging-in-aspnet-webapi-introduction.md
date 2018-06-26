---
date: 2015-01-20T00:00:00Z
description: |
  An introductory post on my new series on how to do paging in ASP.NET Web API. This will look at the various techniques used by some of the most popular APIs
tags:
- aspnet
- aspnet mvc
- aspnet web api
title: 'Paging in ASP.NET Web API: Introduction'
url: /blog/paging-in-aspnet-webapi-introduction/
---

## Introduction

I am starting a new series of posts on performing paging in APIs using ASP.NET Web API. Before I get going I want to introduce you to the various techniques which some of the most popular APIs use. In subsequent posts we will then look at how to implement some of these techniques using ASP.NET Web API. 

So here are some of the varying implementations of pagination used: 

## 1. OData

The first one is probably the easiest to implement, so let's just call this one "lazy man's paging". Or perhaps it should rather be called "clever programmer's paging" since you are saving your client time by not reinventing the wheel, right? It uses the standard OData query operators and allows you to perform paging by adding the OData `$skip` and `$top` query options to any REST API call. 

Mike Wasson has written [a very detailed guide on how to use this on the ASP.NET website](http://www.asp.net/web-api/overview/odata-support-in-aspnet-web-api/supporting-odata-query-options), so I will not spend any more time on this one. And since Mike has done such a good job of describing how to use OData based paging, I will not be going into it any further in this series. If you need a very quick solution to the problem, then this method is at your disposal.

## 2. The Github way

One of the products us developers work most with every day is Github, so let's take a look at how they are doing it by looking at [their official API documentation on pagination](https://developer.github.com/v3/#pagination). Github works with the concept of pages, so to request a specific set of data you will specify the page number you require as well as the number of records per page:

``` text
GET /user/repos?page=4&per_page=2
```

In the example above I requested the list of repositories which belongs to me, stating that I want the 4th page and requested 2 records per page. When the API call returns the list of repostitories in the body of the response as a JSON array:

``` json
[
  {
    "id": 21494724,
    "name": "jerriepelser.com",
    "full_name": "jerriep/jerriepelser.com",
	...
  },
  {
    "id": 17592377,
    "name": "MultiVision",
    "full_name": "jerriep/MultiVision",
	...
  }
]
```

It does not state the total number of records anywhere in the resulting JSON, not does it return the current page or the number of records per page which you requested. What it does however is to return the pagination information in the `Link` header of the request. So in the case of my API call above it returns the following `Link` header:

``` text
Link: <https://api.github.com/user/repos?page=5&per_page=2>; rel="next", 
	  <https://api.github.com/user/repos?page=8&per_page=2>; rel="last", 
	  <https://api.github.com/user/repos?page=1&per_page=2>; rel="first", 
      <https://api.github.com/user/repos?page=3&per_page=2>; rel="prev"
```

So it allows you to request subsequent pages by requesting the URL in the `Link` header with the relation type (`rel` parameter value) of "next". Also to get the total number of pages you will need to extract the `page` query parameter from the link with the relation type of `last`. The GitHub documentation goes into more detail on [how to traverse pagination results](https://developer.github.com/guides/traversing-with-pagination/).

## 3. Pivotal Tracker

The next one I want to demonstrate is how Pivotal Tracker is doing pagination because they actually handle the returning of pagination information in 2 separate ways, depending on how you want it. To request paged information through the Pivotal Tracker API you need to specify the `offset` and `limit` query string parameters (which is different from the "page" based metaphor which GitHub uses), for example:

``` text
GET projects/1/stories?offset=1300&limit=300
``` 

This will return a JSON array containing the projects:

``` json
[
   {
       "id": 1,
		...
   },
   {
       "id": 25,
		...
   },
   {
       "id": 24,
		...
   },
	...
]
```

The actual pagination information is returned as HTTP Headers in the response:

```text
X-Tracker-Pagination-Total: 1543
X-Tracker-Pagination-Limit: 300
X-Tracker-Pagination-Offset: 1300
X-Tracker-Pagination-Returned: 243
```

An alternative approach which the Pivotal Tracker API gives the developer is to [return the response inside an envelope](http://www.pivotaltracker.com/help/api#Envelope_Return_Structure). In this instance the response is wrapped inside a standard envelope, the outer section of which contains certain meta information (such as the pagination information for example), while the original response is returned in the `data` property of the envelope. 

The request the result returned in an envelope, simple specify the value of `true` for the `envelope` query string parameter. So the request:

```text
GET projects/1/stories?offset=1300&limit=300&envelope=true
```    

Will result in the following JSON being returned:

```json
{
   "http_status": "200",
   "data":
	[
	   {
	       "id": 1,
			...
	   },
	   {
	       "id": 25,
			...
	   },
	   {
	       "id": 24,
			...
	   },
		...
	],
   "pagination":
   {
       "total": 1543,
       "limit": 300,
       "offset": 1300,
       "returned": 243
   }
}
```

## 4. Paging in Twitter

Twitter's paging is a bit different from the techniques we have seen so far. Since most data obtained through the Twitter API gets updated on a fairly constant basis they cannot deal with something as simplistic as a page number and a page size (or numerical offsets) to fetch next pages. Rakhitha Nimesh has written [a very detailed article explaining the challenges when dealing with pagination in real time data APIs](http://www.sitepoint.com/paginating-real-time-data-cursor-based-pagination/) over on the Sitepoint blog.

Twitter actually use two different techniques. The first method is used when requesting tweets, and uses the IDs of the tweets to request newer or older items. The second method it uses is cursors. 

### ID based navigation

For time fetching timeline items, Twitter uses minimum and maximum ID based navigation. The guys over at Twitter explains very well how this works, so please read their post on [Working with Timelines](https://dev.twitter.com/rest/public/timelines).

So for Twitter, to do the initial request for a user's timeline, you can make the following request:

``` text
GET /statuses/user_timeline.json?count=5
```

In the example above we requested 5 items and let us say the the following JSON array of tweets was returned:

``` json
[
	{
		"id": 100,
		...
	},
	{
		"id": 99,
		...
	},
	{
		"id": 98,
		...
	},
	{
		"id": 97,
		...
	},
	{
		"id": 96,
		...
	}
]
```

No form of paging information is sent back in either the JSON payload or the HTTP headers, so to page you will need to use the minimum and maximum IDs of the existing tweets you have retrieved to request either older or newer tweets. 

To select older tweets, you will use the `max_id` query string parameter to specify maximum ID you are interested in, and the API will return all tweets with IDs equal or lower to the maximum ID. So given the result returned above we take the lowest ID we have (i.e. 96), subtract 1 from that (which is 95) and tell Twitter that this is the maximum ID we are interested in. Twitter will then return tweets with that ID or lower:

``` text
GET /statuses/user_timeline.json?count=5&max_id=95
```

To return any newer tweets, we will pass the `since_id` parameter of the higest ID we have. Twitter will then return all tweets with IDs higher than that:

``` text
GET /statuses/user_timeline.json?count=5&since_id=100
```

### Cursor based navigation

The other form of navigation which Twitter uses is cursor based navigation. For more information read the article on Twitter entitled [Using cursors to navigate collections](https://dev.twitter.com/overview/api/cursoring). For the cursor based APIs your initial request will be void of any paging information, so make a simple request to the endpoint:

``` text
GET /followers/list.json
```

The JSON object returned by Twitter in this instance contains a `users` property which is the array of followers, but it also contains cursor information to assist in returning next or previous pages of information.

``` json
{
	"users": [
		{
			id": 119228939,
			...
		},
		{
			id": 2376852241,
			...
		},
		...
	],
	"next_cursor": 1486500056211591690,
	"next_cursor_str": "1486500056211591690",
	"previous_cursor": 0,
	"previous_cursor_str": "0"
} 
```

In the example returned above, the `previous_cursor` is zero, which means there are no earlier pages (i.e. we are on the first page), but to request the following page of information we pass in the value of the `next_cursor` property as the `cursor` parameter in the query string of subsequent requests:

``` text
GET /followers/list.json?cursor=1486500056211591690
```

## Conclusion

This was a very quick introduction of some of the paging schemes out in the wild. I have covered only a few examples but their are many more ways in which other APIs tackle this problem. Most of them are however some sort of adaptation of one of the above methods. 

Some of them use paged based paging, while some of the use offsets. Some of them uses some sort of cursor based structure, while some of them use a time or ID based reference value to request newer or older items. As far as the result is concerned, so of them will return no specific paging information, some of them will return it in HTTP header, while some other ones will wrap the result in some sort of envelope which contains meta information such as the pagination information. 

In the next posts in this series I will demonstrate how to implement some of these paging mechanisms in ASP.NET Web API.
