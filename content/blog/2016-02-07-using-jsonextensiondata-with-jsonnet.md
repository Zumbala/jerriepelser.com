---
date: 2016-02-07T00:00:00Z
description: |
  The JSON.NET JsonExtensionData attribute allows you to serialize elements of a JSON document which does not have matching properties to a collection.
tags:
- dotnet
- json.net
title: Using JsonExtensionData with JSON.NET
url: /blog/using-jsonextensiondata-with-jsonnet/
---

## Background

One of the issues we faced when developing the .NET SDK for Auth0 was that user profiles can have different properties based on the origin of the user profile. There are a set of [standard properties](https://auth0.com/docs/user-profile/normalized) for a user profile, but over and above that many of the providers can "enhance" the user profile with extra properties.

Here is an example of a user profile for a database user:

![](/assets/images/using-jsonextensiondata-with-jsonnet/database-user-details.png)

And here is one for a user profile from LinkedIn:

![](/assets/images/using-jsonextensiondata-with-jsonnet/linkedin-user-details.png)

The LinkedIn image is cropped and does not display nearly all the available fields in the screenshot. There are just too many of them. But as you can see the LinkedIn user has a bunch of extra properties which were populated that are unique to LinkedIn and which will most probably not ever be present in profiles from other providers.

This also surfaces when retrieving the user profile through the [Management API](https://auth0.com/docs/api/v2). 

Here is the JSON returned for the database user:

``` json
{
   "email":"jerrie@jerriepelser.com",
   "email_verified":true,
   "user_id":"auth0|568a366de8e57fe426d23100",
   "picture":"https://s.gravatar.com/avatar/6222081fd7dcea7dfb193788d138c457?s=480&r=pg&d=https%3A%2F%2Fcdn.auth0.com%2Favatars%2Fje.png",
   "nickname":"jerrie",
   "identities":[
      {
         "user_id":"...",
         "provider":"auth0",
         "connection":"Username-Password-Authentication",
         "isSocial":false
      }
   ],
   "updated_at":"2016-01-22T07:36:32.419Z",
   "created_at":"2016-01-04T09:07:57.302Z",
   "name":"jerrie@jerriepelser.com",
   "last_password_reset":"2016-01-17T16:50:27.407Z",
   "last_ip":"154.119.56.252",
   "last_login":"2016-01-22T07:36:32.418Z",
   "logins_count":16,
   "blocked_for":[

   ]
}
```

And once again for the LinkedIn user:

``` json
{
   "email":"jerrie@jerriepelser.com",
   "given_name":"Jerrie",
   "family_name":"Pelser",
   "picture":"https://media.licdn.com/mpr/mprx/0_zVjRUCq9dLg1h2I8vUxXU3rUdCZr82R8UYdkU31k9Xuhxe2hMpDczTbEF74fiIVuqJ0LNQDBKd3H",
   "name":"Jerrie Pelser",
   "apiStandardProfileRequest":{
      "headers":{
         "_total":1,
         "values":[
            {
               "name":"x-li-auth-token",
               "value":"..."
            }
         ]
      },
      "url":"https://api.linkedin.com/v1/people/ORklSDbtFm"
   },
   "currentShare":{
      ... removed for brevity ...
   },
   "distance":0,
   "headline":"Software Developer, Blogger, Screencaster, Curator of ASP.NET Weekly.",
   "industry":"Information Technology and Services",
   "location":{
      "country":{
         "code":"th"
      },
      "name":"Thailand"
   },
   "numConnections":331,
   "numConnectionsCapped":false,
   "positions":{
      "_total":4,
      "values":[
         {
            ...
         },
         {
            ...
         },
         {
            ...
         },
         {
            ...
         }
      ]
   },
   "publicProfileUrl":"https://www.linkedin.com/in/jerriepelser",
   "relationToViewer":{
      "distance":0
   },
   "siteStandardProfileRequest":{
      "url":"......",
   "email_verified":true,
   "updated_at":"2016-01-14T14:33:25.720Z",
   "user_id":"linkedin|ORklSDbtFm",
   "nickname":"jerrie",
   "identities":[
      {
         "provider":"linkedin",
         "user_id":"...",
         "connection":"linkedin",
         "isSocial":true
      }
   ],
   "created_at":"2016-01-14T14:33:25.720Z",
   "last_ip":"197.229.128.5",
   "last_login":"2016-01-14T14:33:25.716Z",
   "logins_count":1,
   "blocked_for":[

   ]
}
```

Once again in the case of the LinkedIn document, I have removed a lot of the information returned from the API for brevity. But you get the idea that the API returns vastly different information depending on the original source of the user profile.

For the .NET SDK we wanted to return a strongly typed `User` object but obviously this left us with a decision to make:

1. The first option was to simply return the properties from the [normalized user profile](https://auth0.com/docs/user-profile/normalized) and then somehow make the other properties available dynamically
2. The second option was to return a specialized `User` class for each user profile, depending on the original source. So for a user from LinkedIn we could for example return a `LinkedInUser` object which had all the extra properties for LinkedIn profiles.
3. The last option was to simply add every possible property to the `User` class.

The last option was quickly discounted, and after some deliberation we decided that the second option was also potentially too much work, as it meant that every time the core API team added an extra provider or decided to retrieved extra attributes from a specific provider, that we had to update the SDK as well.

So finally we settled on the first option, but we were still not sure how to do this.

After some investigation I stumbled across the [JsonExtensionData attribute](http://www.newtonsoft.com/json/help/html/T_Newtonsoft_Json_JsonExtensionDataAttribute.htm) in JSON.NET. What this allows you to do is to serialize elements of a JSON document which does not have matching properties on the destination object to the dictionary which is decorated with the `[JsonExtensionData]` attribute. (Also see the JSON.NET documentation on the various [Serialization Attributes](http://www.newtonsoft.com/json/help/html/SerializationAttributes.htm))

## Example

As a practical example, let us assume the following JSON array is being returned from an API call:

```json
[
  {
    "first_name": "Jerrie",
    "last_name": "Pelser",
    "initials": "JJ",
    "profile_image": "http://www.gravatar.com/some_image"
  },
  {
    "first_name": "Peter",
    "last_name": "Parker",
    "initials": "P",
    "profile_image": "http://www.gravatar.com/another_image",
    "address": {
      "city": "New York",
      "suburb": "Forest Hills"
    },
    "family":
    [
      {
        "first_name": "May",
        "last_name": "Parker"
      }
    ]
  }
]
```

We are trying to deserialize this into the following `User` class:

```csharp
public class User
{
    [JsonProperty("first_name")]
    public string FirstName { get; set; }

    [JsonProperty("last_name")]
    public string LastName { get; set; }

    [JsonProperty("initials")]
    public string Initials { get; set; }

    [JsonProperty("profile_image")]
    public string ProfileImage { get; set; }
}
```

The standard properties for the `User` class is `FirstName`, `LastName`, `Initials` and `ProfileImage`. These are common across all our user objects.

As you can see however, the second object in the JSON array has extra attributes which have no backing attributes on our `User` class, so what we can do is to update the `User` class by adding an `AdditionalData` property of type `IDictionary<string, JToken>` which is decorated with the `[JsonExtensionData]` attribute:

```csharp
public class User
{
    [JsonProperty("first_name")]
    public string FirstName { get; set; }

    [JsonProperty("last_name")]
    public string LastName { get; set; }

    [JsonProperty("initials")]
    public string Initials { get; set; }

    [JsonProperty("profile_image")]
    public string ProfileImage { get; set; }

    [JsonExtensionData]
    public IDictionary<string, JToken> AdditionalData { get; set; }
}
```

Now when we deserialize the JSON array, all the extra attributes for the JSON documents which are not mapped to properties in the class will be added to the `AdditionalData` dictionary. 

Below you can see a screenshot of the Visual Studio debugger at runtime, and you can see the extra attributes being added to the dictionary:

![](/assets/images/using-jsonextensiondata-with-jsonnet/runtime-debugger.png)

##  Conclusion

JSON.NET is very easy to use to handle the most common cases, but what makes it so powerful is that it also has a few really advanced features which allows you to control how data gets serialized and deserialized, so you can almost always find a way to work around those tricky edge cases.



