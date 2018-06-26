---
date: 2013-03-20T00:00:00Z
description: |
  Demonstrates how you can subscribe to the collection change notifications in ReactiveUI to keep to sepate collections in sync.
tags:
- mvvm
- reactiveui
- winrt
title: Synchronising collections using ReactiveUI
url: /blog/synchronising-collections-using-reactiveui/
---

With the [announcement ](http://blog.paulbetts.org/index.php/2013/03/12/reactiveui-4-5-is-released/)of ReactiveUI 4.5 and the fact that it now works with the Xamarin products as well, I have decided to give it a try and see if it gives me better cross platform capabilities than Caliburn Micro (which has no support for the Xamarin products at all).  My first task was to try and get my head around Reactive Extensions, and for that I found the website [Introduction to Rx](http://www.introtorx.com/) extremely useful. Rx is still messing with my head though and I suspect it will for some time to come.

As I am working through some of my existing ViewModel classes to convert them to ReactiveUI I also try and see if I can find better ways to do certain things.  One of the scenarios I have is that I have a list of accounts which is displayed on the screen.  The user can then select certain of the accounts which I will keep track of in a separate list of selected accounts.

``` csharp
public ObservableCollection<AccountViewModel> Accounts
{
    get { return accounts; }
}

public ObservableCollection<AccountViewModel> SelectedAccounts
{
    get { return selectedAccounts; }
}
```

The user is also allowed to do maintenance on the list of accounts which means that items can be added to or removed from the list of accounts.  Of course if an item gets removed from the list of accounts it must also be removed from the list of selected accounts otherwise it will not be valid anymore.  For his purpose I wrote the following piece of code to do some basic synchronisation between the two collections:

``` csharp
Accounts.CollectionChanged += (sender, args) =>
{
    if (args.Action == NotifyCollectionChangedAction.Remove)
    {
        foreach (AccountViewModel accountViewModel in args.OldItems)
        {
            SelectedAccounts.Remove(accountViewModel);
        }
    }
    else if (args.Action == NotifyCollectionChangedAction.Reset)
        SelectedAccounts.Clear();
};
```

With ReactiveUI I first of all changed the collections from ObservableCollection<T> to ReactiveCollection<T>:

``` csharp
public ReactiveCollection<AccountViewModel> Accounts
{
    get { return accounts; }
}

public ReactiveCollection<AccountViewModel> SelectedAccounts
{
    get { return selectedAccounts; }
}
```

And changed the collection synchronisation code the use the observable pattern:

``` csharp
Accounts.ItemsRemoved.Subscribe(x => SelectedAccounts.Remove(x));
Accounts.CollectionCountChanged.Subscribe(x =>
{
    if (x == 0)
        SelectedAccounts.Clear();
});
```

Unit tests pass and all still works as expected.  And I am one step closer to making more sense of Reactive Extensions