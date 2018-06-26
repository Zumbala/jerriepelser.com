---
date: 2013-02-25T00:00:00Z
description: |
  Demonstrates how to configure the ViewLocator and ViewModelLocator in Caliburn Micro when you split your Views and View Models across different assemblied.
tags:
- caliburn micro
- mvvm
- winrt
- xaml
title: Split Views and ViewModels in Caliburn Micro
url: /blog/split-views-and-viewmodels-in-caliburn-micro/
---

As mentioned in my previous [blog post](/blog/passing-custom-parameters-to-caliburn-micro-actions/) I am developing a Windows Store application using [Caliburn Micro](http://caliburnmicro.codeplex.com/) (CM) in which I have the views and view models split into different assemblies.  CM is largely convention based and therefore expect things is certain places.  One of these conventions is the way in which CM locates views and view models.  Let's say that we have an application with a root namespace of MyApplication and our views are located in the namespace MyApplication.Views.  The CM convention is that it will expect the view models to be located in the same assembly in the namespace MyApplication.ViewModels.  Working outside of this convention will cause CM to be unable to find the view models for your view and vice versa.

Luckily for us CM allows us to override these default conventions with our own and tell the framework where to locate our views and view models.  For the purposes of this blog post I have created an application which consists of two assemblies.  The first is called Client and is a Windows Store application which contains among other things my views in the namespace Client.Views.  The second is a Class Library called Common and contains my view models in the namespace Common.ViewModels.

![](/assets/images/2013/02/Untitled.png)

Informing Caliburn Micro where to locate our views and view models in this case consists of  two parts.  The first part is to tell it what the namespaces are for the views and view models, and the second part is to tell it in which assemblies the views and view models can be located.

For the first part we will need to configure the type mappings for the ViewLocator and ViewModelLocator.  This is done in the Configure method of our App class:

``` csharp
var config = new TypeMappingConfiguration
{
    DefaultSubNamespaceForViews = "Client.Views",
    DefaultSubNamespaceForViewModels = "Common.ViewModels"
};
ViewLocator.ConfigureTypeMappings(config);
ViewModelLocator.ConfigureTypeMappings(config);
```

In the code above we create a new instance of the TypeMappingConfiguration class and override the values of the DefaultSubNamespaceForViews and DefaultSubNamespaceForViewModels properties.  The default values for these two properties are "Views" and "ViewModels" respectively.  Note that as the name of the properties suggest we only need to specify the sub-namespaces, or rather the parts of the namespaces which differ.  If we for example had our views in a namespace called "MyApp.Views" and our view models in a namespace called "MyApp.Logic.ViewModels", because they both have the part of the namespace called "MyApp" in common we would only need to specify the parts which differ:

``` csharp
var config = new TypeMappingConfiguration
{
    DefaultSubNamespaceForViews = "Views",
    DefaultSubNamespaceForViewModels = "Logic.ViewModels"
};
ViewLocator.ConfigureTypeMappings(config);
ViewModelLocator.ConfigureTypeMappings(config);
```

In our example however there are no common parts to the namespaces so we specify the complete namespaces.

The second part we need to configure is to tell Caliburn Micro in which assemblies to locate the views and view models.  By default it will look in the same assembly where the App class is located, in this case the assembly called Client.  To tell it to look at additional assemblies we simply override the SelectAssemblies() method of our App class to add the assembly which contains our view models to the list of assemblies:

``` csharp
protected override IEnumerable<Assembly> SelectAssemblies()
{
    var assemblies = base.SelectAssemblies().ToList();
    assemblies.Add(typeof(MainViewModel).GetTypeInfo().Assembly);

    return assemblies;
}
```

With that very basic and simple configuration done we can now develop our application and keep our views and view models in separate assemblies.
