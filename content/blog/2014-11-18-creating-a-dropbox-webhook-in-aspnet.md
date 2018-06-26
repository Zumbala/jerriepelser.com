---
date: 2014-11-18T00:00:00Z
description: |
  Demonstrates how to create a Dropbox webhook in ASP.NET and ensure that requests to the webhook originates from Dropbox
tags:
- webhook
- aspnet
- dropbox
title: Creating a Dropbox Webhook in ASP.NET
url: /blog/creating-a-dropbox-webhook-in-aspnet/
---

Dropbox provides a mechanism for your application to be notified when a user's files changes through a concept of webhooks. This means that you do not have to poll a user's Dropbox account at certain intervals to check for changes. You simply specify the webhook and when you get notified of changes to a user's files you can call the `/delta` API method in Dropbox to get the list of actual changes. The functionality around webhooks is fairly [well explained in the Dropbox documentation](https://www.dropbox.com/developers/webhooks/docs). They even provide a [nice sample tutorial in their documentation](https://www.dropbox.com/developers/webhooks/tutorial).

The problem however is that the tutorial is in Python and I could not find a nice example of how to do this in C# with the HMAC-SHA256 signature. So here is the nice example of how to do this in C# and ASP.NET :)

#Handling the verification
As stated in the Dropbox Webhooks documentation, your webhook needs to handle 2 requests. The first is a verification request which is done when you add the webhook in the [App Console](https://www.dropbox.com/developers/apps). The second is the actual notification which gets sent when users of you application make changes to their Dropbox.

Handling the verification call is super simple. Simply create an MVC controller with an Action which takes a `challenge` parameter and returns that parameter back:

```csharp
public class WebhooksController : Controller
{
    [HttpGet]
    public ActionResult Dropbox(string challenge)
    {
        return Content(challenge);
    }
}
```

#Handling notifications
The the notications is a little bit more complex as you want to verify the signature of the request to ensure that the request originated from Dropbox, and if not to ignore it. It is not required to do this verification, but not doing so will leave your system open to people sending you fake notification messages.

So the first bit is to write the code which verifies the signature header against the body. The signature contains a HMAC-SHA256 encoded signature of the request body using the App Secret of your Dropbox application as the signing key. So first of I write 2 methods to help me with this. The first is called `GetSha256Hash` and helps me to compute the hash value of an input string. The second is called `VerifySha256Hash` and this takes and input string, computes its hash and then compares that hash with the hashed signature I received from Dropbox to ensure that they match up. 

```csharp
private string GetSha256Hash(HMACSHA256 sha256Hash, string input)
{
    // Convert the input string to a byte array and compute the hash. 
    byte[] data = sha256Hash.ComputeHash(Encoding.UTF8.GetBytes(input));

    // Create a new Stringbuilder to collect the bytes 
    // and create a string.
    var stringBuilder = new StringBuilder();

    // Loop through each byte of the hashed data  
    // and format each one as a hexadecimal string. 
    foreach (byte t in data)
    {
        stringBuilder.Append(t.ToString("x2"));
    }

    // Return the hexadecimal string. 
    return stringBuilder.ToString();
}

private bool VerifySha256Hash(HMACSHA256 sha256Hash, string input, string hash)
{
    // Hash the input. 
    string hashOfInput = GetSha256Hash(sha256Hash, input);

    if (String.Compare(hashOfInput, hash, StringComparison.OrdinalIgnoreCase) == 0)
        return true;

    return false;
}
```

> The code above is adapted from code I found while doing a Google search, but for the life of me I cannot find the original post anymore to give credit where credit is due.

With that in place I can write the code for my second controller action to which Dropbox will do a POST whenever a user updates their Dropbox:

```csharp
public class WebhooksController : Controller
{
    [HttpGet]
    public ActionResult Dropbox(string challenge)
    {
        return Content(challenge);
    }

    [HttpPost]
    public async Task<ActionResult> Dropbox()
    {

        // Get the request signature
        var signatureHeader = Request.Headers.GetValues("X-Dropbox-Signature");
        if (signatureHeader == null || !signatureHeader.Any())
            return new HttpStatusCodeResult(HttpStatusCode.BadRequest);

        // Get the signature value
        string signature = signatureHeader.FirstOrDefault();

        // Extract the raw body of the request
        string body = null;
        using (StreamReader reader = new StreamReader(Request.InputStream))
        {
            body = await reader.ReadToEndAsync();
        }

        // Check that the signature is good
        string appSecret = ConfigurationManager.AppSettings["Dropbox_AppSecret"];
        using (HMACSHA256 hmac = new HMACSHA256(Encoding.UTF8.GetBytes(appSecret)))
        {
            if (!VerifySha256Hash(hmac, body, signature))
                return new HttpStatusCodeResult(HttpStatusCode.BadRequest);
        }

        // Do your thing here... e.g. store it in a queue to process later
		// ...

        // Return A-OK :)
        return new HttpStatusCodeResult(HttpStatusCode.OK);
    }
}
``` 

So looking at the code above the first thing I do is check whether the `X-Dropbox-Signature` header exists, and if it doesn't I return HTTP Status Code 400. Next up I read the actual body of the request from the input stream. I read the Dropbox app secret from my configuration settings and use that, along with the signature header and the request body to verify that this request originated from Dropbox. If not I once again throw a HTTP Status Code 400.

Once the verification has passed you can do whatever you need to do to process the data. **Please note** however that you need to return from the webhook within 10 seconds so it is not advised to do the actual processing of the changes at this point. For my application I simply store the JSON received from the webhook in an Azure Queue and then have a [WebJob](http://azure.microsoft.com/en-us/documentation/articles/web-sites-create-web-jobs/) which processes that queue to retrieve the notifications and call the `/delta` method of the Dropbox API to retrieve the actual changes for each user and do some more processing at that point.

The last bit is to return HTTP Status 200 back to Dropbox to let them know you processed the request successfully.  