---
date: 2015-04-07T00:00:00Z
description: |
  Testing business logic which depends on the current date for calculation can be a bit tricky. I show you how you can mock the current date so you can write better unit tests for this code.
tags:
- unit testing
- mocking
- tdd
title: Unit testing with Dates
url: /blog/unit-testing-with-dates/
---

## Overview

In a recent app we developed, there was a particular piece of code which monitored user accounts and would send out a notification before their subscription expires. There were various sets of notifications which we emailed to them; one a week before expiration, one 3 days before, one the day before expiration, and a final one once the subscription has actually expired.

We needed to write unit tests for all these different scenarios to ensure that the correct emails go out on the correct dates. In a situation like this it is fairly easy to mock the data access layer and email system to simulate certain conditions, but how does one go about simulating a specific date or time?

## Change the system clock

Well, the first option is obviously to mess around with the system clock and set the date to a specific simulated date. 

This is not a good idea... Why? Let me explain.

Firstly there are various pieces of software on your computer depending on the current date and time, and messing around with that can seriously mess things up. Some software licensing services also check whether you are messing around with the date to try and circumvent their licensing, and can lock you out when you do this.

Secondly this is unreliable for a number of reasons:

1. Most computers these days are configured to update the system time automatically from an internet based service. So you may still be thinking that you set the clock to a specific date to try and simulate a certain scenario, and the automatic clock updating procedure could have corrected the time.
2. Certain unit test runners will run unit tests in parallel. So your unit tests may be tripping over each other in changing the system clock.
3. The time code takes to execute may be unpredictable, and the clock keeps ticking while your tests are running. For date-level accuracy you may get away with this, but for code which depends on second or millisecond accuracy this will not work. Let us suppose that you set up your testing scenario and set the clock to a specific date and time (to the second or millisecond level). By the time the code which needs the current time executes, 100 ms could have elapsed. Or maybe 200ms. Or maybe 2 seconds. Point is you do not know, which makes it unreliable.

Bottom line is that you should not be messing around with the system clock for unit tests. Best case is that your unit tests will fail. Worst case is that you will mess up your computer.     

## Passing in the current date

The next option is to pass in the current date to whichever method or class needs it. This is certainly a very valid way to do it.

Let us say that I have a piece of code which checks accounts to send out expiry notifications. Inside the method I have a line of code which gets the current date in order to determine how long it is until the user's account expires, so it can send out the correct notification if required: 

```csharp
public void SendExpiryNotifications()
{
    DateTime currentTime = DateTime.UtcNow;

    // Rest of the code
}
```

A very simple change I can make to this method so I can more easily create certain scenarios for unit testing is to change the `currentTime` variable to a parameter instead;

```csharp
public void SendExpiryNotifications(DateTime currentTime)
{
    // Rest of the code
}
```

Now where I call this method in code I can pass in the current date:

```csharp
SendExpiryNotifications(DateTime.UtcNow);
```  

And for unit tests I can pass in the specific date for the scenario I would like to test:

```csharp
SendExpiryNotifications(new DateTime(2015, 4, 4, 14, 0, 0, DateTimeKind.Utc));
```

## Making the clock a service

Another approach, and the one which I normally use in situations like this, is to have a service which provides the current system time and then use that inside my other services which depends on having access to the current system time. This will typically be something like the following interface:

```csharp
public interface ISystemClock
{
    DateTime GetCurrentTime();
}
```

The actual implementation will return the current system time:

```csharp
public class SystemClock : ISystemClock
{
    public DateTime GetCurrentTime()
    {
        return DateTime.UtcNow;
    }
}
``` 

Then in my code I can use dependency injection to inject it into the class which contains the `SendExpiryNotifications` method. So let us say I have an `ExpiryNotificationService` class, I will pass in `ISystemClock` in the constructor: 

```csharp
public class ExpiryNotificationService
{
    private readonly ISystemClock systemClock;

    public ExpiryNotificationService(ISystemClock systemClock)
    {
        this.systemClock = systemClock;
    }

    public void SendExpiryNotifications()
    {
        // Get the current time
        DateTime currentTime = systemClock.GetCurrentTime();

        // Rest of the code
    }
}
```

At runtime I will use dependency injection to ensure the correct implementation is injected into the constructor, but during testing I can create a mock class, and pass that through to my class under test. Using NSubstitute as my mocking library, a typical unit test will look like this:

```csharp
public class ExpiryNotificationServiceTests
{
    [Fact]
    public void SomeUnitTest()
    {
        // Arrange
        var systemClock = Substitute.For<ISystemClock>();
        systemClock.GetCurrentTime().Returns(new DateTime(2015, 4, 4, 14, 0, 0, DateTimeKind.Utc));

        var service = new ExpiryNotificationService(systemClock);

        // Act
        service.SendExpiryNotifications();

        // Assert
        // ...
    }
}
``` 

## Using Noda Time

The last approach is actually to use a library like [Noda Time](http://nodatime.org/). This is essentially the same as what I did above, but you get all the advantages which Noda Time offers over .NET's normal DateTime libraries (read [Why does Noda Time exist?](http://nodatime.org/1.3.x/userguide/rationale.html)).

NodaTime already comes with an `IClock` interface built in:

```csharp
public interface IClock
{
    /// <summary>
    /// Gets the current <see cref="Instant"/> on the time line according to this clock.
    /// </summary>
    Instant Now { get; }
}
```

You can switch to using Noda Time's `Instant` class, or you can use the `Instant.ToDateTimeUtc()` method to convert back to the normal .NET `DateTime` structure if too much of you code still depends on using that.

Noda Time has an implementation of `IClock` which uses the system clock and which you can use during runtime:

```csharp
public sealed class SystemClock : IClock
{
    /// <summary>
    /// The singleton instance of <see cref="SystemClock"/>.
    /// </summary>
    /// <value>The singleton instance of <see cref="SystemClock"/>.</value>
    public static readonly SystemClock Instance = new SystemClock();

    /// <summary>
    /// Constructor present to prevent external construction.
    /// </summary>
    private SystemClock()
    {
    }

    /// <summary>
    /// Gets the current time as an <see cref="Instant"/>.
    /// </summary>
    /// <value>The current time in ticks as an <see cref="Instant"/>.</value>
    public Instant Now { get { return NodaConstants.BclEpoch.PlusTicks(DateTime.UtcNow.Ticks); } }
}
```   

So let's change the `ExpiryNotificationService` to use Noda Time:

```csharp
public class ExpiryNotificationService
{
    private readonly IClock systemClock;

    public ExpiryNotificationService(IClock systemClock)
    {
        this.systemClock = systemClock;
    }

    public void SendExpiryNotifications()
    {
        // Get the current time
        Instant currentTime = systemClock.Now;

        // Rest of the code
    }
}
```

For unit tests you can then easily mock the `IClock` interface in exactly the same manner I did in the unit testing example above. 

Besides that approach, Noda Time also has a `FakeClock` class which you can use instead of a mock object. Using that instead, your unit test code may look like this:

```csharp
public class ExpiryNotificationServiceTests
{
    [Fact]
    public void SomeUnitTest()
    {
        // Arrange
        var systemClock = new FakeClock(Instant.FromUtc(2015, 4, 4, 14, 0, 0));
        var service = new ExpiryNotificationService(systemClock);

        // Act
        service.SendExpiryNotifications();

        // Assert
        // ...
    }
}
```

The `FakeClock` class also has another nice feature where you set it up to auto advance a certain duration every time you make a call to the `Now` property:

```csharp
var systemClock = new FakeClock(Instant.FromUtc(2015, 4, 4, 14, 0, 0), Duration.FromSeconds(1));
```

In the example above the first time a call is made it to `Now`, it will return 2PM on the 4th of April 2015. The second time it will be 2:01PM, the third time 2:02PM, etc. This is nice when you want simulate time passing inside your unit test.

## Conclusion

In this blog post a demonstrated a few different approaches you can use when unit testing code that depends on the current system date and time. 