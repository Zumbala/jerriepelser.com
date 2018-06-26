---
date: 2014-02-18T00:00:00Z
description: |
  ASP.NET Identity 2.0 adds support for adding 2 factor authentication to your application. This walks you throught the process of setting it up.
tags:
- 2 factor authentication
- aspnet
- aspnet identity
- aspnet mvc
title: 2 Factor authentication with ASP.NET Identity 2.0 - Beta 1
url: /blog/2-factor-authentication-with-asp-net-identity-2-0-beta-1/
---

A while ago I was looking at doing a blog post about 2 Factor authentication in ASP.NET MVC using Google Authenticator.  I then became aware that it was on the roadmap for the ASP.NET Identity team to add to ASP.NET Identity 2.0\.  Well, [ASP.NET Identity 2.0 Beta 1 is now available](http://blogs.msdn.com/b/webdev/archive/2014/02/11/announcing-preview-of-microsoft-aspnet-identity-2-0-0-beta1.aspx) with 2 Factor Authentication, so in this blog post I will give you a quick overview on how it works and how to implement an email provider for the 2 Factor Authentication.

The default ASP.NET templates will probably be updated in due course, but in the meantime here are the steps to get ASP.NET Identity 2.0 working in a **new** application:

First you create a new ASP.NET Application using File | New Project and selecting an **ASP.NET Web Application**

![](/assets/images/2014/02/Capture.png)

The second step is to update all the Nuget packages in the project.  Ensure that you include prerelease packages so you can get the ASP.NET Identity 2.0 packages.

![](/assets/images/2014/02/Untitled.png)

Next up you can install the ASP.NET Identity Samples package to make the required changes to your project for using 2 Factor Authentication.  Once again ensure that you have prerelease packages enabled and search for and install the package "Microsoft.AspNet.Identity.Samples".

![](/assets/images/2014/02/Capture1.png)

**Please note: **There seems to be a problem with the current ASP.NET Identity samples package in that it adds a second connection string to you web.config.  To fix that just open your web.config file and remove the second connection string which was added.

At this point you can run the application to test the 2 Factor Authentication.  The only problem is that you will need to implement the Email and SMS services to get the messages delivered to a user.  So let's get going on implementing the Email service.

## Implementing the Email Service using the standard SMTP Client

The email service is used in a 3 places from what I can figure out so far: when registering to confirm the email address, when the user forgot their password and also when 2 factor authentication is enabled it is used to deliver the verification code to the user.

The existing implementation which does nothing can be found in the file IdentityConfig.cs

``` csharp
public class EmailService : IIdentityMessageService
{
    public Task SendAsync(IdentityMessage message)
    {
        // Plug in your email service here to send an email.
        return Task.FromResult(0);
    }
}
```

In a production system you will probably want to use a transactional email service such as SendGrid, Mandrill or Mailgun but for now I will just use the standard SMTP Client provided as part of the .NET Framework and configure it to drop the email in a file location instead of actually sending the mail.

First I implement SendAsync() method of the EmailService class

``` csharp
public class EmailService : IIdentityMessageService
{
    public Task SendAsync(IdentityMessage message)
    {
        // Create the mail message
        var mailMessage = new MailMessage(
            "myapp@myapp.com",
            message.Destination,
            message.Subject,
            message.Body
            );

        // Send the message
        SmtpClient client = new SmtpClient();
        client.SendAsync(mailMessage, null);

        return Task.FromResult(true);
    }
}
```

And I also update the web.config file to drop the email in a folder instead of actually sending the mail message.  Of course you are welcome to configure an SMTP server.

``` xml
<system.net>
    <mailSettings>
        <smtp deliveryMethod="SpecifiedPickupDirectory">
            <specifiedPickupDirectory pickupDirectoryLocation="C:\temp\MailPickup"/>
        </smtp>
    </mailSettings>
</system.net>
```

## Testing the Email Service

To test the mail service you can start off by running the application and registering a new user.

![](/assets/images/2014/02/Capture2.png) 

After registration you will be prompted with a screen informing you that you need to confirm the mail address.  To do that you can either click on the link which is provided on that page for demo purposes, or click on the link in the email which was sent.  Since I have implemented the email provider I browse to the location where the email was dropped and open it in Outlook.

![](/assets/images/2014/02/Capture3.png)

Once you click on the link in the email your email address will be confirmed and you can log in to the application using the email address and password supplied during registration.

## Enabling 2 Factor Authentication

To enable 2 Factor Authentication on a user's account you can go to the management page for a user by clicking on the email address on the top right corner of the application after you have logged in:

![](/assets/images/2014/02/Untitled2.png)

You will be presented with a page where you can update certain user settings.  Click on the link to enable 2 Factor Authentication.

![](/assets/images/2014/02/Untitled1.png)

To test the 2 Factor Authentication you can log out of the application and log in again.  After logging in you will be prompted to select the 2 factor authentication provider.  Since we have not yet added or confirmed a telephone number the only option available is **EmailCode**. Click on the Submit button to continue.**
**

![](/assets/images/2014/02/Untitled3.png)

You will receive an email containing the code.

![](/assets/images/2014/02/Capture21.png)

Enter the verification code received in the email and click Submit to continue.

![](/assets/images/2014/02/Capture4.png)

## Conclusion

In this blog post I gave an overview of the new 2 Factor Authentication in ASP.NET Identity 2.0 and demonstrated how to implement an email service which is used for account confirmation as well as to deliver confirmation codes to users for 2 Factor Authentication.