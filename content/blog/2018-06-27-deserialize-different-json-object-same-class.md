---
title: "Deserializing different JSON structures to the same C# class"
description:
  The JsonConverter class allows you to deserialize differently structured JSON objects to the same C# class.
tags:
- .net
- json.net
---

## Introduction

During the development of the [Auth0 .NET SDK](https://github.com/auth0/auth0.net), I came across an interesting scenario where some of the API endpoints in the [Auth0 Management API](https://auth0.com/docs/api/management/v2) returned differently structured JSON objects.

For example, some of the API endpoints returned the following JSON payload when an error occurred:

```json
{
   "statusCode": 400,
   "error": "Error",
   "errorCode": "error_code",
   "message": "The Message",
}
```

While other API endpoints returned the following JSON payload for errors:

```json
{
   "statusCode": 400,
   "name": "Error",
   "code": "error_code",
   "description": "The Message"
}
```

When any API endpoint returns an error, I want to throw an exception. For this purpose, I created an `ApiException` class which also contains an instance of an `ApiError` class:

```csharp
public class ApiError
{
    [JsonProperty("error")]
    public string Error { get; set; }

    [JsonProperty("errorCode")]
    public string ErrorCode { get; set; }

    [JsonProperty("message")]
    public string Message { get; set; }

    [JsonProperty("statusCode")]
    public int StatusCode { get; set; }
}

public class ApiException : Exception
{
    public ApiError ApiError { get; }

    //...
}
```

The idea is that the error payload returned from the API endpoint is deserialized into the `ApiError` instance so the developer can have access to that information. You will notice that the `JsonProperty` attributes on the properties correlate with the first JSON payload I listed above.

_But how do I deserialize the second JSON payload into an instance of that same class?_

## Enter JsonConverter

Thankfully JSON.NET allows for a lot of flexibility in how JSON objects are deserialize. I have written two [blog](/blog/custom-converters-in-json-net-case-study-1/) [posts](/blog/using-jsonextensiondata-with-jsonnet/) on this before.

To handle this particular scenario, I created a `JsonConverter` with a simple mapping table which maps the attributes of the second JSON payload to the attributes of the first JSON payload.

The `ReadJson` method retrieves all the properties on the class it is being mapped to - in this case, the `ApiError` class. Then it loops through the properties (or attributes) of the JSON object being deserialized and for each property it checks to see whether there is a mapping in the `_propertyMappings` dictionary.

If there is a mapping, it uses the mapped property name instead. This means, for example, that if it comes across a property in the JSON object named `code`, it will use the property name `errorCode` instead.

Then it finds the property on the C# class which has a `JsonProperty` attribute with that name. So in the example of the JSON property `errorCode`, this will return the `ErrorCode` property of the `ApiError` class.

Then it sets the value of the C# property to the value of the JSON property.

```csharp
public class ApiErrorConverter : JsonConverter
{
    private readonly Dictionary<string, string> _propertyMappings = new Dictionary<string, string>
    {
        {"name", "error"},
        {"code", "errorCode"},
        {"description", "message"}
    };

    public override bool CanWrite => false;

    public override void WriteJson(JsonWriter writer, object value, JsonSerializer serializer)
    {
        throw new NotImplementedException();
    }

    public override bool CanConvert(Type objectType)
    {
        return objectType.GetTypeInfo().IsClass;
    }

    public override object ReadJson(JsonReader reader, Type objectType, object existingValue, JsonSerializer serializer)
    {
        object instance = Activator.CreateInstance(objectType);
        var props = objectType.GetTypeInfo().DeclaredProperties.ToList();

        JObject jo = JObject.Load(reader);
        foreach (JProperty jp in jo.Properties())
        {
            if (!_propertyMappings.TryGetValue(jp.Name, out var name))
                name = jp.Name;

            PropertyInfo prop = props.FirstOrDefault(pi =>
                pi.CanWrite && pi.GetCustomAttribute<JsonPropertyAttribute>().PropertyName == name);

            prop?.SetValue(instance, jp.Value.ToObject(prop.PropertyType, serializer));
        }

        return instance;
    }
}
```

I also need to add the `JsonConverter` attribute to the `ApiError` class to instruct JSON.NET to use my `ApiErrorConverter` class for serialization and deserialization.

```csharp
[JsonConverter(typeof(ApiErrorConverter))]
public class ApiError
{
    //...
}
```

## Testing it out

To test this out, I created a simple C# console application which takes the two different JSON payloads and tries to deserialize them. Both payloads have essentially the same values but use different attributes names.

```csharp
class Program
{
    static void Main(string[] args)
    {
        string payload1 = "{\"statusCode\":400,\"error\":\"Error\",\"errorCode\":\"error_code\",\"message\":\"The Message\"}";
        string payload2 = "{\"statusCode\":400,\"name\":\"Error\",\"code\":\"error_code\",\"description\":\"The Message\"}";

        var apiError1 = JsonConvert.DeserializeObject<ApiError>(payload1);
        var apiError2 = JsonConvert.DeserializeObject<ApiError>(payload2);
    }
}
```

When I run the application in debug mode and inspect the `apiError1` and `apiError2` variables, you can see that they both have the same values, meaning that I was able to successfully deserialize the two different JSON payloads to the same class:

![Values of variables in Watch window](/images/blog/2018-06-27-deserialize-different-json-object-same-class/watch.png)