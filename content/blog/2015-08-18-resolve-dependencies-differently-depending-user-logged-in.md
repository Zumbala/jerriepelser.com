---
date: 2015-08-18T00:00:00Z
description: |
  I demonstrate how you can use Autofac to resolve dependencies differently based on whether the user is authenticated or not.
tags:
- dependency injection
- autofac
title: Resolve dependencies differently depending on whether user is logged in
url: /blog/resolve-dependencies-differently-depending-user-logged-in/
---

In a recent project I had to collect information from a user, and store that information so that we have the information available during the entire browsing session. 

The application allowed for users to use the application anonymously, in which case I would store the information in a session variable. If a user was however logged into the application, I would store the information inside a database table so that it would be available on the next visit.

At first I wrote a single class which handled both cases by checking whether the user was logged in, and then either store or retrieve the information from the session or database.

I then thought of another way to handle this, and instead of doing the logic inside a single class, I split it out into two different classes and depending on whether the user is logged in I would use the appropriate class.

First I created a generic interface which specifies the contract for the class:

``` csharp
public class ServiceLink
{
    public ServiceLink(string serviceType, string accountName)
    {
        ServiceType = serviceType;
        AccountName = accountName;
    }

    public string AccountName { get; set; }
    public string ServiceType { get; set; }
}

public interface IServiceLinkStorage
{
    Task Add(ServiceLink serviceLink);
    Task<ServiceLink> Get(string serviceType);
    Task<List<ServiceLink>> GetAll();
    Task Remove(string serviceType);
    Task Clear();
}
```    

And created two separate implementations. One for storing the information inside a session variable, and the other inside the database:

``` csharp
public class ServiceLinkSessionStorage : IServiceLinkStorage
{
	...
}

public class ServiceLinkDatabaseStorage : IServiceLinkStorage
{
	...
}
```

I used Autofac to inject `IServiceLinkStorage` into my controllers, so the only thing left was to resolve the correct class based on whether the user was logged in or not:

``` csharp
builder.Register<IServiceLinkStorage>(context =>
   {
       var httpContext = context.Resolve<HttpContextBase>();

       if (httpContext != null && httpContext.User.Identity.IsAuthenticated)
           return new ServiceLinkDatabaseStorage(httpContext, context.Resolve<IApplicationDbContext>());

       return new ServiceLinkSessionStorage(context.Resolve<HttpSessionStateBase>());
   })
.As<IServiceLinkStorage>()
.InstancePerRequest();
```

Obviously you will need to use a lifetime scope of `InstancePerDependency()` or `InstancePerRequest()` for this to work correctly.

How would you have handled this situation? Please share in the comments below.  