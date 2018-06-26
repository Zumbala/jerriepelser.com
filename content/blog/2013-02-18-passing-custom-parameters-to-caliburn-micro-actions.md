---
date: 2013-02-18T00:00:00Z
description: |
  Demonstrates how you can pass custom parameters to Caliburn Micro actions by using the SpecialValues collection of the MessageBinder class.
tags:
- caliburn micro
- mvvm
- winrt
- xaml
title: Passing custom parameters to Caliburn Micro actions
url: /blog/passing-custom-parameters-to-caliburn-micro-actions/
---

I have been working with [Caliburn Micro](http://caliburnmicro.codeplex.com/) (CM) for the past month, using it in a new Windows 8 application I am busy developing.  CM is largely convention based but it does allow you to override and customize a lot of those conventions as well as provide other points of extensibility.

## The initial solution

I recently had a scenario where I had a ListView displaying the a list of items.  When a used clicked on one of the items in the ListView, a method on the underlying ViewModel was called using the [CM Action mechanism](http://caliburnmicro.codeplex.com/wikipage?title=All%20About%20Actions&referringTitle=Documentation "All%20About%20Actions&referringTitle=Documentation").  I needed to access the model for the item which was clicked on as I needed to perform some actions on that.  Looking at the [CM documentation on Actions](http://caliburnmicro.codeplex.com/wikipage?title=All%20About%20Actions&referringTitle=Documentation "All%20About%20Actions&referringTitle=Documentation") you will notice that there are a number of arguments which you can pass to the method which is executed, of of which being **$eventArgs**. For the click event on a ListView this will be of type [ItemClickEventArgs](http://msdn.microsoft.com/en-us/library/windows/apps/windows.ui.xaml.controls.itemclickeventargs).  From there it is as simple as inspecting the ClickedItem property of the event arguments as that will contain my underlying ViewModel item.

``` csharp
public void TakeActionOnEventArgs(ItemClickEventArgs args)
{
    AccountViewModel account = args.ClickedItem as AccountViewModel;

    if (account != null)
    {
        var dialog = new MessageDialog(string.Format("You clicked {0}", account.Name), "Using $eventArgs");
        dialog.ShowAsync();
    }
}
```

And to hook the method up to my view I declare the XAML for the ListView as follows:

``` xml
<ListView
    x:Name="accountsListView1"
    AutomationProperties.AutomationId="AccountsListView1"
    AutomationProperties.Name="Accounts1"
    TabIndex="1"
    Margin="0,10,0,0"
    Padding="10,0,0,60"
    ItemsSource="{Binding Accounts}"
    ItemTemplate="{StaticResource Standard80ItemTemplate}"
    SelectionMode="None"
    IsItemClickEnabled="True"
    caliburn:Message.Attach="[Event ItemClick] = [TakeActionOnEventArgs($eventArgs)]"
    IsSwipeEnabled="false" />
```

Notice in the XAML above that I use the Message.Attach syntax, which will then be parsed by CM and will execute the TakeActionOnEventArgs() method passing along the event  arguments to the method.  Also notice for my purposes I needed to set the IsItemClickEnabled property to true, as it is set to false by default.

## A platform independent solution

Normally the solution above will suffice, but for my particular application I could not allow platform specific code to bleed into my view models, as the application will eventually need to run on multiple platforms (Windows Store apps, Windows Phone, etc).

Reading further through the CM documentation I noticed that the **$eventArgs** is simply a "special value" which is understood by the parser and that CM allow us to specify our own special values using the SpecialValues collection of the MessageBinder class.  In the Configure() method of my Application I specified the following code to define my own special value called **$account**.

``` csharp
MessageBinder.SpecialValues.Add("$account", context =>
{
    if (context == null || context.EventArgs == null)
        return null;

    return
        ((ItemClickEventArgs)context.EventArgs).ClickedItem as AccountViewModel;
});
```

The method in my ViewModel is also changed to take an AccountViewModel as parameter.

``` csharp
public void TakeActionOnAccount(AccountViewModel account)
{
    if (account != null)
    {
        var dialog = new MessageDialog(string.Format("You clicked {0}", account.Name), "Using $account");
        dialog.ShowAsync();
    }
}
```

And finally the markup in my XAML is also changed to bind the **$account** parameter to the Action Message.

``` xml
<ListView
    x:Name="accountsListView2"
    AutomationProperties.AutomationId="AccountsListView2"
    AutomationProperties.Name="Account21"
    TabIndex="1"
    Margin="0,10,0,0"
    Padding="10,0,0,60"
    ItemsSource="{Binding Accounts}"
    ItemTemplate="{StaticResource Standard80ItemTemplate}"
    SelectionMode="None"
    IsItemClickEnabled="True"
    caliburn:Message.Attach="[Event ItemClick] = [TakeActionOnAccount($account)]"
    IsSwipeEnabled="false" />
```

**Please note** that the scenario above is very simplistic for demonstration purposes and may not make sense as everything is located in one assembly.  In my proper application however the view models is located in a completely separate assembly which is not platform specific and can then be reused by all the platform specific assemblies by simply linking the one shared file into the platform specific assemblies.  This is similar to what is described in an [article](http://docs.xamarin.com/guides/cross-platform/application_fundamentals/building_cross_platform_applications/sharing_code_options) in the Xamarin Developer Centre.  The code which defines the **$account** special value is located in the platform specific assembly and will differ between the various platforms.