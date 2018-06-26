---
date: 2013-07-08T00:00:00Z
description: |
  My first attempt at submitting an app to the Windows 8 app store went less than spectacular. Here are a few things which you need to look out for.
tags:
- facebook
- windows store
- winrt
title: 'App Certification Chronicles: Facebook Accounts and the In-App Purchase'
url: /blog/app-certification-chronicles-test-accounts/
---

## Submitting to the Windows Store

I recently went through the process of submitting my application, [One Love](http://www.oneloveapp.com), to the Windows Store.  I read through the [certification requirements](http://msdn.microsoft.com/en-us/library/windows/apps/hh694083.aspx) and ran my app through the [Certification Kit](http://msdn.microsoft.com/en-us/library/windows/apps/hh694081.aspx), so I was pretty sure that there wouldn't be any problems.  Low and behold, my application was rejected.  The testers referred me specifically to section 1.2 of the requirements, relating to test accounts.

[One Love](http://www.oneloveapp.com) is an application which allows you to update your status on all your social media accounts at the same time.  Obviously I could not give the testers my Facebook account details, so when I first submitted the application I made a note to the testers that they should just use their own Facebook or other social media account to test the application.  I was therefore a bit surprised when the application was rejected because of this reason.  I resubmitted the application again, making it more clear to the testers that for me to create a second  account on Facebook for testing purposes would be against the Terms and Conditions of Facebook so they must please use their own account to test.

The application came back as rejected the second time around as well.  The issue was again regarding the test account, but this time there was a second issue relating to the in-app purchase experience not working correctly.  In turned out that even after a person concluded an in-app purchase, the option for them to purchase would still be available in my application.  This was easy enough to fix, so I did it quickly.

## Facebook Test Accounts

The second issue of the Facebook test accounts remained, and I was not quite sure what to do regarding this.  Surely I could not have been the first person to experience this issue, so I put my friend Google to work.  It turns out Facebook does indeed support the creation of test users.  Here is how you go about creating legal test users:

Navigate to your Facebook application settings, and under the **Settings** section, select **Developer Roles**.[![2013-07-03 01-47-56 PM](/assets/images/2013/07/2013-07-03-01-47-56-PM.png)](/assets/images/2013/07/2013-07-03-01-47-56-PM.png)

You will be presented with a list of all the administrators, developers and testers of your application.  The last section on the list is for test users.  According to the Facebook documentation, a test user is "_a special user account, invisible to "normal" users, which is created on behalf of an app for the purpose of testing that app's Facebook integration_".

To create a test user, click on the **Create** link in the Test Users section.  Facebook will prompt you for how many test users you want to create.  Select the number of user you want to create and click on the **Create** button.

![](/assets/images/2013/07/2013-07-03-01-53-56-PM.png)

You should see the newly created users listed under the Test Users section.  To set a password for the user you can click on Set Password next to the name of the user.  This will allow you to authenticate the user through the normal OAuth authentication process which my application uses to interface with the Facebook API.

![](/assets/images/2013/07/2013-07-03-01-56-16-PM2.png)

To view the email address for a test user you can click on the **Modify** link in the Test Users section.  This will bring up a list of all the test users.  Their email addresses are listed in one of the columns:

![](/assets/images/2013/07/2013-07-03-02-02-53-PM1.png)

Simply give this email address and the password you created earlier to the testers who have to approve your application for the Windows Store. They can use it to log into Facebook from within your application as well as log into Facebook itself to ensure that whatever interaction your application has with Facebook is working correctly.

Though my experience so far is only with the Windows Store, I am sure that submission to the Apple App Store and Google Play Store will have similar requirements.

Hope this helps someone else out there.
