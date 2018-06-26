---
date: 2013-10-15T00:00:00Z
description: |
  Sometimes it is necessarry to mock an abstract class. I demonstrate how you can achieve this using NSubstitute as your mocking framework.
tags:
- mocking
- mocks
- tdd
- unit tests
title: Mocking an abstract class using NSubstitute
url: /blog/mocking-abstract-class-using-nsubstitute/
---

For the development of [One Love](http://www.oneloveapp,com), I make use of the wonderful [MvvmCross](https://github.com/MvvmCross/MvvmCross) library for implementing the [MVVM](http://en.wikipedia.org/wiki/Model_View_ViewModel) pattern.  This allows me to have a LOT of shared code across all platforms, and also allows me to implement unit testing across all that shared code.  There are of course certain things which are platform specific and for that I use the dependency injection which comes standard with MvvmCross.  For the unit tests I use a mocking framework to create mocks which are then passed into the constructors of the classes under test.

One of the aspects which is different on each platform is licensing and in-app purchases.  My requirement for licensing so far is very simple.  I basically want to know whether a user has made an in-app purchase for a certain feature, and I also want to allow a user to actually perform the in-app purchase.  There is however some other logic around the licensing service which is shared.  In the free version of [One Love](http://www.oneloveapp.com), you can add a single Twitter, Facebook or LinkedIn account.  Once you have added your allotted single account for each service, you cannot add another unless you make the in-app purchase for the professional version.

This is what my licensing service interface looks like:

``` csharp
public interface ILicensingService
{
    bool HasLicenseForFeature(string featureName);

    Task PurchaseLicenseForFeature(string featureName);

    bool IsSharingClientAvailable(ISharingClient sharingClient, IEnumerable<SharingAccount> currentAccounts);
}
```

The code for the IsSharingClientAvailable() method is shared across all platforms.  It does however depend on the HasLicenseForFeature() method which is platform specific.  So what I wanted to do is to create an abstract class which performs the shared logic and can then be inherited on each platform to implement the platform specific code to interface with the relevant app store on that platform.  I also of course wanted to be able to unit test that shared code, even though some methods of the class under test needed to be mocked.

My current mocking framework of choice is [NSubstitute](http://nsubstitute.github.io/), and it allows me to do exactly that.  It does however come with certain caveats, so make sure you understand them by reading the ["Creating a substitute"](http://nsubstitute.github.io/help/creating-a-substitute/) section of the documentation.

So understanding that I created an abstract licensing service class with the shared logic:

``` csharp
public abstract class BaseLicensingService : ILicensingService
{
    public abstract bool HasLicenseForFeature(string featureName);

    public abstract Task PurchaseLicenseForFeature(string featureName);

    public bool IsSharingClientAvailable(ISharingClient sharingClient, IEnumerable<SharingAccount> currentAccounts)
    {
        string[] standardServices =
        {
            "facebook",
            "twitter",
            "linkedin"
        };

        if (!HasLicenseForFeature("Pro"))
        {
            if (!standardServices.Contains(sharingClient.Id))
                return false;
            else if (currentAccounts.Any(a => a.Service == sharingClient.Id))
                return false;
        }

        return true;
    }
}
```

And for my unit test I simply mock out HasLicenseForFeature() method to simulate the behaviour I want to test:

``` csharp
[TestClass]
public class BaseLicensingServiceTests
{
    private BaseLicensingService licensingService;
    private ISharingClient sharingClient;
    private List<SharingAccount> currentAccounts = new List<SharingAccount>();

    [TestInitialize]
    public void SetUp()
    {
        licensingService = Substitute.For<BaseLicensingService>();
        sharingClient = Substitute.For<ISharingClient>();
    }

    [TestMethod]
    public void WhenFacebookAndProShouldBeAvailable()
    {
        // Arrange
        licensingService.HasLicenseForFeature("Pro").Returns(true);
        sharingClient.Id.Returns("facebook");

        // Act
        bool isAvailable = licensingService.IsSharingClientAvailable(sharingClient, currentAccounts);

        // Assert
        isAvailable.Should().BeTrue();
    }

    [TestMethod]
    public void WhenFacebookAndNotProShouldBeAvailable()
    {
        // Arrange
        licensingService.HasLicenseForFeature("Pro").Returns(false);
        sharingClient.Id.Returns("facebook");

        // Act
        bool isAvailable = licensingService.IsSharingClientAvailable(sharingClient, currentAccounts);

        // Assert
        isAvailable.Should().BeTrue();
    }

    [TestMethod]
    public void WhenFacebookAndNotProWithAccountsShouldNotBeAvailable()
    {
        // Arrange
        licensingService.HasLicenseForFeature("Pro").Returns(false);
        sharingClient.Id.Returns("facebook");
        currentAccounts.Add(new SharingAccount(Guid.Empty, "Name", null, "facebook", "ServiceId", "ServiceType", "Token"));

        // Act
        bool isAvailable = licensingService.IsSharingClientAvailable(sharingClient, currentAccounts);

        // Assert
        isAvailable.Should().BeFalse();
    }
}
```

I am sure a lot of architectural purists will point out other ways this should rather be done, but it works for me.  Hopefully you may also find this feature of NSubstitute useful for your own unit tests.