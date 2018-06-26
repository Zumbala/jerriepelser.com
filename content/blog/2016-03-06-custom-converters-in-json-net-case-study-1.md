---
date: 2016-03-06T00:00:00Z
description: |
  Custom converters in JSON.NET allow you to write custom logic when serializing or deserializing JSON. In this post I look at an example where the JSON returned could be either an object or an array.
tags:
- dotnet
- json.net
title: 'Using custom converters in JSON.NET: Array or Object?'
url: /blog/custom-converters-in-json-net-case-study-1/
---

## The problem

During the development of the [Auth0 .NET SDK](https://github.com/auth0/auth0.net), I ran into an issue with one of our [Management API](https://auth0.com/docs/api/v2) calls where it could return a different JSON structure based on parameters passed in by the user. 

The offending API call in question was the [Users endpoint](https://auth0.com/docs/api/v2#!/Users/get_users) where you could pass in a parameter called `include_totals` which will return the list of users, along with the total number of records and some other paging information. If you did not pass in this parameter (or specifeied a value of `false`), it would simply return an array of users.

To make this a bit more visual, here is an approximate example when you as the API call to include the totals:

``` json
{
   "start":0,
   "limit":50,
   "length":36,
   "total":36,
   "users":[
      {
         "email":"john.doe@gmail.com",
         "email_verified":false,
         "username":"johndoe",
         "phone_number":"+199999999999999",
         "phone_verified":false,
         "user_id":"usr_5457edea1b8f33391a000004",
         "created_at":"",
         "updated_at":"",
         "identities":[
            [
               {
                  "connection":"Initial-Connection",
                  "user_id":"5457edea1b8f22891a000004",
                  "provider":"auth0",
                  "isSocial":false
               }
            ]
         ],
         "app_metadata":{

         },
         "user_metadata":{

         },
         "picture":"",
         "name":"",
         "nickname":"",
         "multifactor":[
            ""
         ],
         "last_ip":"",
         "last_login":"",
         "logins_count":0,
         "blocked":false
      },
      {

      },
      {

      }
   ]
}
```

So you can see that the API returns a JSON object with the paging information, and then a `users` property containing an array with the list of users.

In the case where you ask to not include the totals, the API would simply return an array of users, e.g.:

``` json
[
   {
      "email":"john.doe@gmail.com",
      "email_verified":false,
      "username":"johndoe",
      "phone_number":"+199999999999999",
      "phone_verified":false,
      "user_id":"usr_5457edea1b8f33391a000004",
      "created_at":"",
      "updated_at":"",
      "identities":[
         [
            {
               "connection":"Initial-Connection",
               "user_id":"5457edea1b8f22891a000004",
               "provider":"auth0",
               "isSocial":false
            }
         ]
      ],
      "app_metadata":{

      },
      "user_metadata":{

      },
      "picture":"",
      "name":"",
      "nickname":"",
      "multifactor":[
         ""
      ],
      "last_ip":"",
      "last_login":"",
      "logins_count":0,
      "blocked":false
   },
   {

   },
   {

   }
]
```

In my case I wanted to always return the user an instance of the `PagedList<User>` class, which is defined as follows:

``` csharp
public class PagedList<T> : List<T>
{
    public PagedList()
    {
    }

    public PagedList(IEnumerable<T> collection) : base(collection)
    {
    }

    public PagedList(IEnumerable<T> collection, PagingInformation paging) : base(collection)
    {
        Paging = paging;
    }

    public PagedList(int capacity) : base(capacity)
    {
    }

    public PagingInformation Paging { get; set; }
}

public class PagingInformation
{
    [JsonProperty("length")]
    public int Length { get; set; }

    [JsonProperty("limit")]
    public int Limit { get; set; }

    [JsonProperty("start")]
    public int Start { get; set; }

    [JsonProperty("total")]
    public int Total { get; set; }

    public PagingInformation(int start, int limit, int length, int total)
    {
        Start = start;
        Limit = limit;
        Length = length;
        Total = total;
    }
}
```

Depending on whether they request paging information, the `Paging` property will contain the paging information.

## JsonConverter to the rescue

As you can expect this creates problem when trying to deserialize the resulting JSON to a specific .NET type, because we are working with two very different potential JSON structures being deserialized.

Thankfully JSON.NET offers a solution by allowing you to create a custom converter which specifies how an object is serialized or deserialized.  All you need to do is inherit from `JsonConverter` and then provide implementations for the `CanConvert`, `WriteJson` and `ReadJson` methods.

The logic then is fairly simple; I simply check whether the JSON being serialized is an object or an array. 

If it is an object, I know that the user requested the totals, so I extract the various paging information properties, and deserialize the "users" property of the JSON object to a list of `User`. I then return a `PagedList<User>` with all the list of users as well as the paging information.

In the case where the JSON is an array, I know that paging information was not requested, so I simply convert the JSON to a list of `User`, and return a `PagedList<User>` with the list of users and no paging information:

``` csharp
internal class UserPagedListConverter : JsonConverter
{
    public override void WriteJson(JsonWriter writer, object value, JsonSerializer serializer)
    {
        throw new System.NotImplementedException();
    }

    public override bool CanConvert(Type objectType)
    {
        return typeof (PagedList<User>).GetTypeInfo().IsAssignableFrom(objectType.GetTypeInfo());
    }

    public override object ReadJson(JsonReader reader, Type objectType, object existingValue, JsonSerializer serializer)
    {
        if (reader.TokenType == JsonToken.StartObject)
        {
            JObject item = JObject.Load(reader);

            if (item["users"] != null)
            {
                var users = item["users"].ToObject<IList<User>>(serializer);

                int length = item["length"].Value<int>();
                int limit = item["limit"].Value<int>();
                int start = item["start"].Value<int>();
                int total = item["total"].Value<int>();

                return new PagedList<User>(users, new PagingInformation(start, limit, length, total));
            }
        }
        else
        {
            JArray array = JArray.Load(reader);

            var users = array.ToObject<IList<User>>();

            return new PagedList<User>(users);
        }

        // This should not happen. Perhaps better to throw exception at this point?
        return null;
    }
}
```

Simple as that. Then when you want to deserialize an JSON string, you can simply pass along the `UserPagedListConverter` to the `Convert` method, e.g.

``` csharp
JsonConvert.DeserializeObject<PagedList<User>>(content, new UserPagedListConverter());
```
