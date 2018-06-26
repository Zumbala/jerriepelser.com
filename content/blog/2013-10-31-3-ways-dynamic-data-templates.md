---
date: 2013-10-31T00:00:00Z
description: |
  Demonstrates how you can use Value Converters, Data Template Selectors and the Visual State Manager to make your Windows 8.1 GridView data templates more dynamic.
tags:
- datatemplate
- datatemplateselector
- winrt
- xaml
title: 3 Techniques you can use to make your data templates dynamic
url: /blog/3-ways-dynamic-data-templates/
---

One of the nice things of the .NET Framework is that there are so many different ways to achieve the same result.  I admit that it can sometimes make things a bit confusing, but in general I like the fact there are a lot of different ways to solve the same problem.

I ran into this again recently when I wanted to make the data templates I use in a Windows 8 GridView more "dynamic", i.e. I wanted the content to display differently based on the underlying data.  I figured out 3 different techniques to achieve more or less the same result, and actually use all 3 of them in [my application](http://www.oneloveapp.com) for different reasons.

In this blog post I am going to discuss and demonstrate each of these techniques, namely:

1.  Value Converters
2.  Visual State Manager
3.  Data Template Selectors
Seeing that I love travelling I took some data for this blog post from the [Geckos Adventures](http://www.geckosadventures.com/) website for trips in South East Asia.  Some trips are on sale so I want the trip data in the GridView to display a bit differently for trips which are on sale:

![](/assets/images/2013/10/Untitled6.png)

The first image represents a normal trip - I simply display an image for the trip, as well as the trip name and price.  The second image is for a trip which is on sale.  In this case I want to strikeout the normal price and display the discounted price and I also want to display an green label at the bottom indicating it is a special promotion.  It is not going to win any design contest, I know, but it will do for the purpose of this blog post.

## 1. Value Converters

The first and probably the quickest way you can do it is by using a value converter.  I use a value converter which converts the boolean "IsSale" property of my underlying data model to a [Visibility](http://msdn.microsoft.com/EN-US/library/windows/apps/windows.ui.xaml.visibility(v=vs.10).aspx) enumeration.  First is to create a very simple **BooleanToVisibilityConverter** which takes boolean value and returns Visibility.Visible if true, and Visibility.Collapsed if false:

``` csharp
public class BooleanToVisibilityConverter : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter, string language)
    {
        return (value is bool && (bool)value) ? Visibility.Visible : Visibility.Collapsed;
    }

    public object ConvertBack(object value, Type targetType, object parameter, string language)
    {
        return value is Visibility && (Visibility)value == Visibility.Visible;
    }
}
```

You can then use this converter in your binding markup to toggle the visibility of certain elements based on whether or not the item is on sale:

``` xml
<Border x:Name="border" BorderThickness="1"
    HorizontalAlignment="Right" VerticalAlignment="Bottom"
    Background="Lime"
    Visibility="{Binding IsSale, Converter={StaticResource BooleanToVisibilityConverter}}">
    <TextBlock>
        <Run Text="Special!!"/>
    </TextBlock>
</Border>
```

### When would you use Value Converters?

I would suggest using this technique when the sort of interactivity you want to achieve is as simple as perhaps toggling the visibility of a couple of elements.  You can write more complex value converters to achieve all sorts of results, but when the state changes become quite involved this may become a bit of a nightmare to maintain or even to understand the markup later on, as you will need to understand not just the markup, but also what all the value converters are doing in the background.

## 2. Visual State Manager

The second way is to use the VisualStateManager and Behaviors in Blend for Visual Studio 2013\.  As a primer you can go and read Jeremy Nixon's blog post entitled [Everything I know about Behaviors in Blend for Visual Studio 2013](http://blog.jerrynixon.com/2013/10/everything-i-know-about-behaviors-in.html).

First of all, because we are using the Visual State Manager you need to extract the data template to a user control.  The rest is pretty simple.  Create the visual states with their storyboards:

``` xml
<VisualStateManager.VisualStateGroups>
    <VisualStateGroup x:Name="SaleStates">
        <VisualState x:Name="IsSale">
            <Storyboard>
                ...
            </Storyboard>
        </VisualState>
    <VisualState x:Name="IsNotSale"/>
</VisualStateGroup>
</VisualStateManager.VisualStateGroups>
```

The transition between the different state are triggered using behaviors:

``` xml
<interactivity:Interaction.Behaviors>
    <core:DataTriggerBehavior Binding="{Binding IsSale}" Value="True">
        <core:GoToStateAction StateName="IsSale"/>
    </core:DataTriggerBehavior>
    <core:DataTriggerBehavior Binding="{Binding IsSale}" Value="False">
        <core:GoToStateAction StateName="IsNotSale"/>
    </core:DataTriggerBehavior>
</interactivity:Interaction.Behaviors>
```

And finally the data template is as simple as displaying the custom user control with the states and behavior we created above:

``` xml
<DataTemplate x:Key="VisualStateManagerTemplate">
    <local:TripDisplayControl />
</DataTemplate>
```

### When would you use the Visual State Manager?

First of all I would suggest using this technique when the difference between the different states involves changing the states of a large number of elements and when the change between states become more involved that simply setting the visibility of a few elements.  Maybe you want to move elements to a different position, resize them, play around with the opacity, etc.  To me it just seems cleaner to do this in a storyboard.  Of course the fact that you are using storyboards also opens up another nice possibility in that you can animate the transitions between different states, so I would definitely look into this technique in scenarios like that.

## 3. Data Template Selector

The final technique we can use is by making use of a [Data Template Selector](http://msdn.microsoft.com/en-us/library/windows/apps/windows.ui.xaml.controls.datatemplateselector.aspx).  This is very handy is scenarios where the difference between the data displayed in the different visual states is so big that you want to display completely different data templates.  A Data Template Selector allows you to apply logic at runtime to specify which data template to display.

First off you create the different data templates:

``` xml
<DataTemplate x:Key="SaleDataTemplateTemplate">
...
</DataTemplate>
<DataTemplate x:Key="NotSaleDataTemplateTemplate">
...
</DataTemplate>
```

Then create a data template selector class which inherits from [DataTemplateSelector](http://msdn.microsoft.com/EN-US/library/windows/apps/windows.ui.xaml.controls.datatemplateselector(v=vs.10).aspx) and override the [SelectTemplateCore()](http://msdn.microsoft.com/EN-US/library/windows/apps/br209472(v=vs.10).aspx) method to implement your logic.  In the case of our example I simply check the value of the IsSale property of the underlying data object, and based on that I return the correct data template.

``` csharp
public class TripTemplateSelector : DataTemplateSelector
{
    public DataTemplate SaleTemplate { get; set; }
    public DataTemplate NotSaleTemplate { get; set; }
    protected override DataTemplate SelectTemplateCore(object item, DependencyObject container)
    {
        var sale = item as Trip;
        if (sale != null && sale.IsSale)
            return SaleTemplate;
        return NotSaleTemplate;
    }
}
```

The next step is to declare the new data template selector as a resource:

``` xml
<Page.Resources>
    <common:TripTemplateSelector x:Name="TripTemplateSelector" 
        SaleTemplate="{StaticResource SaleDataTemplateTemplate}"
        NotSaleTemplate="{StaticResource NotSaleDataTemplateTemplate}"
        />
</Page.Resources>
```

And finally you specify the data template selector on your GridView instead on a data template:

``` xml
<GridView 
    ItemsSource="{Binding Trips}"
    ItemTemplateSelector="{StaticResource TripTemplateSelector}"
    SelectionMode="None"
    />
```

### When would you use a data template selector?

My example for using a data template selector is a bit simplistic.  This technique would ideally be used when the difference between the various data templates is significant enough that trying to maintain a single data template for the various scenarios does not make sense.  In such cases it is easier to just create completely separate data templates and use a data template selector to supply the correct data template at runtime.

The data template selector also allows you to get access to the container in which the data template is hosted, and this opens up a whole slew of other possibilities.  In my application [One Love](http://www.oneloveapp.com) for example I will disallow selection of items in certain scenarios and for this I use a data template selector in which I inspect the data model and based on that will disable a grid view item which does not allow people to select it at all.  Below is a short snippet of how I achieve that:

``` csharp
public class AddAccountDataTemplateSelector : DataTemplateSelector
{
    public DataTemplate AvailableTemplate { get; set; }
    public DataTemplate UnavailableTemplate { get; set; }
    protected override DataTemplate SelectTemplateCore(object item, DependencyObject container)
    {
        var viewModel = item as AccountViewModel;
        var selectorItem = container as SelectorItem;
        if (selectorItem != null && viewModel != null)
        {
            if (viewModel.IsLocked)
            {
                selectorItem.IsEnabled = false;
                return UnavailableTemplate;
            }
        }
        return AvailableTemplate;
    }
}
```

## Conclusion

This blog post demonstrated a few different techniques which you can use do make the data display of your WinRT GridView a bit more flexible.  However simple or complex your requirements, you should be able to find one of these 3 techniques useful for your own data display needs.  You also do not use the techniques in isolation but can even use all 3 of them together at the same time.

The data used in this demo was hijacked from the [Geckos Adventures](http://www.geckosadventures.com/) website.