---
date: 2014-02-24T00:00:00Z
description: |
  Shows how you can send the emails for the 2 factor authentication in your ASP.NET MVC application via Twilio.
tags:
- 2 factor authentication
- aspnet
- aspnet identity
- aspnet mvc
- twilio
title: 2 Factor Authentication via SMS with ASP.NET Identity 2.0 and Twilio
url: /blog/2-factor-authentication-via-sms-asp-net-identity-2-0-twilio/
---

In my [previous blog post](/blog/2-factor-authentication-with-asp-net-identity-2-0-beta-1/) I demonstrated how to send 2 Factor Authentication security codes via email in the new ASP.NET Identity 2.0 Beta 1\.  In this blog post I will demonstrate how to send the 2 Factor security codes via SMS by using the [Twilio](https://www.twilio.com/) API. This blog post will build on the previous one, so if anything in this post is unclear please refer to that blog post first

This blog post is not a tutorial on using Twilio with the .NET Framework, so I will assume that you have some level of familiarity with using the Twilio C# client.  The official Twilio documentation does have a nice [.NET Quickstart](https://www.twilio.com/docs/quickstart/csharp) which will help you get going if you are not familiar with it.

## Implementing the SMS Service

You may recall from my previous blog post that the templates created by the ASP.NET Identity 2.0 Samples package creates placeholders for both an email and an SMS service which can be used to deliver 2 Factor Authentication security codes.  This is what the default SmsService class which is located in App_Start\IdentityConfig.cs looks like:

``` csharp
public class SmsService : IIdentityMessageService
{
    public Task SendAsync(IdentityMessage message)
    {
        // Plug in your sms service here to send a text message.
        return Task.FromResult(0);
    }
}
```

To create the implementation which sends the security code via SMS we will need to install the Twilio C# client via Nuget first.  Locate the Nuget package called **Twilio** and install it:

![](/assets/images/2014/02/Capture5.png)]

Once you have the Twilio package installed, you can do the actual implementation by adding 2 lines of code:

``` csharp
public class SmsService : IIdentityMessageService
{
    public Task SendAsync(IdentityMessage message)
    {
        TwilioRestClient client = new TwilioRestClient("<Your Account SID>", "<Your account auth token>");
        client.SendSmsMessage("<The number you are sending from>", message.Destination, message.Body);

        return Task.FromResult(0);
    }
}
```

Be sure to add pass your own Account SID and Auth Token which you will find in your Twilio Dashboard into the constructor.  Also pass in one of your telephone numbers as the "from" parameter of the SendSmsMessage() method call.

## Enable your telephone number

To use your phone number for 2 Factor Authentication you will need to add it to your user account first.  After you have logged in to the application, head on over to the Account Management page and click on the Add link next to "Phone Number". **Please note** that you will also need to be sure that 2 Factor Authentication is enabled.

![](/assets/images/2014/02/Capture11.png)

Clicking on the **Add** link you will take you to a page where you can add your telephone number.  Enter your phone number and click on **Send verification code**.

![](/assets/images/2014/02/Capture22.png)

You will receive an SMS message containing the verification code which is sent to the phone number which you are trying to add.

![](/assets/images/2014/02/IMG_0728.png)

Enter the verification code on the web page and click **Submit**.

![](/assets/images/2014/02/Capture311.png)

After entering the verification code you will be returned to the Account Maintenance page where the new phone number will be listed.

![](/assets/images/2014/02/Capture41.png)

## Log in using your phone number

After you have confirmed the new phone number you can use it to receive your 2 Factor Authentication security code when logging in.  To test this make sure you log out of the application and log in to the application again.  After logging in using your email address and password you will be presented with a page where you can specify where you want to send the 2 Factor Authentication security code to.  We have added both an email address and phone number we can send it to either, but for the purposes of this demo we will select **PhoneCode** in order to send an SMS to our phone number.

![](/assets/images/2014/02/Capture51.png)

You will receive the SMS with the security code.

![](/assets/images/2014/02/IMG_0730.png)

Enter the security code on the web page and click **Submit** to log in to the application.

![](/assets/images/2014/02/Capture6.png)

## Conclusion

In this blog post we build on the previous post to add 2 Factor Authentication via SMS using the Twilio API.  Thanks to the Twilio Nuget package it is as simple as adding two lines of code.