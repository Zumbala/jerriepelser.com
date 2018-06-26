---
date: 2013-02-21T00:00:00Z
description: |
  When setting the DataContext property of a XAML page in a Windows 8 Store app, Caliburn Micro does not set the DataContext to the correct View Model.
tags:
- caliburn micro
- mvvm
- winrt
- xaml
title: 'Caliburn Micro: Beware the default Windows Store app templates'
url: /blog/caliburn-micro-beware-the-default-windows-store-app-templates/
---

I was recently dumbfounded by an apparent issue with Caliburn Micro in a Windows Store application being unable to locate the view model for a view.  The symptoms was that the application navigated correctly to my view, but the data from the view model was not being displayed on the page.

What complicated matters a bit more was that my views and view models were located in different assemblies and non-default namespaces, therefore I immediately suspected the problem was located there.  After some debugging I determined that Caliburn Micro was indeed locating and instantiating the correct view model for my view, but somehow the view model was not bound to the view.

My application was using the view first approach, so on application launch the following code was executed.

``` csharp
protected override void OnLaunched(LaunchActivatedEventArgs args)
{
    DisplayRootView<MainView>();
}
```

I decided to switch it around to the view model first approach by changing my OnLaunched() method and all of a sudden it worked.

``` csharp
protected override void OnLaunched(LaunchActivatedEventArgs args)
{
    DisplayRootViewFor<MainViewModel>();
}
```

I knew that was not a proper solution and did some more digging.  The culprit turned out to be the fact that the templates for Windows Store applications in Visual Studio sets the `DataContext` of a page to the `DefaultViewModel` property of the `LayoutAwarePage` class and somehow CM does not set the DataContext of a page if it has already been set.

![](/assets/images/2013/02/Untitled1.png)

The solution turned out to be as simple as deleting the offending line of markup and after that Caliburn Micro was setting the DataContext correctly and the data was displayed on the page.

There is a discussion on the Caliburn Micro website which [describes this issue and the reason behind it](http://caliburnmicro.codeplex.com/discussions/404719).