---
date: 2013-11-05T00:00:00Z
description: |
  Demonstrates how you can disable selection of Windows 8.1 GridView items dynamically by using a DataTemplateSelector
tags:
- windows store
- winrt
- xaml
title: How to disable item selection in the WinRT GridView
url: /blog/disable-item-selection-winrt-gridview/
---

When I sat down to convert [One Love](http://www.oneloveapp.com) to Windows 8.1 I decided to rework the way in which I present the selection of social networks when a user adds a new account.  In the free version you can add a single personal profile each for Facebook, Twitter and LinkedIn.  If you wanted to add LinkedIn Groups for example you would have to upgrade to the Professional version (which is an in-app purchase).  In the old version of One Love I would however not even present the availability of accounts which are not available in the free version.  In the new version I wanted to display to the user that these accounts are available, but the user must not be able to select them.  I also wanted to present them with some sort of indication that says "hey, you can add me but upgrade to the Pro version first", as is depicted in the screenshot below.

![](/assets/images/2013/11/Untitled8.png)

So how would you go about doing this?  Remember from my [previous blog post](/blog/3-ways-dynamic-data-templates/) I discussed 3 ways in which you can handle data templates in the XAML GridView.  Under the section on data template selectors, I mentioned in passing that you can also access the container for a data item which allows you to disable the container which effectively disables selection for that item.

So, let us run step by step through the process of creating something similar to what I have done in the screenshot above.

## Create the Data Templates

Seeing as I am using a DataTemplateSelector, am going to create 2 data templates.  One for the available state, and one for the unavailable state

``` xml
<DataTemplate x:Key="AccountSelectionListViewItemTemplate">
    <Grid Width="320" Height="100" HorizontalAlignment="Stretch">
        <!-- See source code on Github for the rest of the template  -->
    </Grid>
</DataTemplate>

<DataTemplate x:Key="AccountSelectionListViewItemTemplateUnavailable">
    <Grid Width="320" Height="100" HorizontalAlignment="Stretch">
        <ContentControl Width="320" Height="100"
                        ContentTemplate="{StaticResource AccountSelectionListViewItemTemplate}"
                        DataContext="{Binding}" />

        <!-- Overlay everything with a big-ass store logo -->
        <Grid Grid.Column="0" Background="#CC000000">
            <Viewbox x:Name="PurchaseViewbox" Width="40" Height="40" Margin="20,-10,0,0"
                        HorizontalAlignment="Left" VerticalAlignment="Center">
                <Grid HorizontalAlignment="Center" VerticalAlignment="Center">
                    <Path Width="40" Height="40" Margin="0,0,0,0"
                            Data="M20.698999,37.680001L20.728297,47.16587 33.61,48.981002 33.61,37.680001z M10.143999,37.680001L10.143999,45.847396 20.156999,47.165 20.156999,37.680001z M20.025199,27.401999L10.275801,28.895499 10.1,37.151998 20.113,37.151998z M33.638999,25.660998L20.728,27.286056 20.728,37.107999 33.638999,37.107999z M44.969753,12.589L54.221,15.750469 54.221,60.485732 45.08625,63.999999 0,55.216289 0,20.904119z M26.158921,4.0209999L26.395999,5.3842993C24.468168,6.2397809,22.635324,7.6746707,22.837091,10.019763L22.837091,15.251596 21.43,15.512 21.43,9.0262496C21.43,9.0262494,21.570728,5.7488637,26.158921,4.0209999z M32.518852,2.9860001C32.518852,2.9860001,37.235877,3.4820871,37.649997,8.4481478L37.649997,12.513213 36.325736,12.758 36.325736,8.0340929C36.325736,8.0340924 35.829513,3.8128014 30.532497,4.2268629 30.532497,4.2268629 30.39635,4.2437935 30.162599,4.2828627L29.982999,3.1474352C30.757728,3.0575933,31.597007,3.0003119,32.518852,2.9860001z M23.714635,0C23.714635,0,28.721195,0.52604294,29.159999,5.7970142L29.159999,14.083626 27.899585,14.316731 27.755082,14.316731 27.755082,5.356904C27.755082,5.3569036 27.227678,0.87761402 21.606511,1.3164406 21.606511,1.3164406 12.999011,2.3711443 13.438517,7.4650126L13.438517,16.989993 11.944999,17.265999 11.944999,6.4116182C11.944999,6.4116182,12.208602,0.17447281,23.714635,0z"
                            Stretch="Uniform" Fill="White" RenderTransformOrigin="0.5,0.5">
                        <Path.RenderTransform>
                            <TransformGroup>
                                <TransformGroup.Children>
                                    <RotateTransform Angle="0" />
                                    <ScaleTransform ScaleX="1" ScaleY="1" />
                                </TransformGroup.Children>
                            </TransformGroup>
                        </Path.RenderTransform>
                    </Path>
                </Grid>
            </Viewbox>
            <TextBlock Margin="80,0,10,10" HorizontalAlignment="Stretch" VerticalAlignment="Center"
                        FontSize="14" Foreground="White" TextWrapping="Wrap">
                To link this account, purchase the Professional Version
            </TextBlock>
        </Grid>
    </Grid>
</DataTemplate>
```

The first data template called **AccountSelectionListViewItemTemplate** specifies the layout of the item which is available for selection and displays the image, name and service type. The second data template named **AccountSelectionListViewItemTemplateUnavailable** is used for items which are not available for selection.  Instead on repeating the entire layout from the first data template, I reuse the first template by creating a ContentControl with the **ContentTemplate** property specified as the first template, and then overlay all of that with the store logo and purchase message on a semi transparent dark background so that a little bit of the underlying account details displays through.

## Create the DataTemplateSelector

Now that the two data templates are done, we need to create a DataTemplateSelector which will supply the correct data template at runtime.

``` csharp
public class AccountDataTemplateSelector : DataTemplateSelector
{
    public DataTemplate AvailableTemplate { get; set; }
    public DataTemplate UnavailableTemplate { get; set; }

    protected override DataTemplate SelectTemplateCore(object item, DependencyObject container)
    {
        var viewModel = item as Account;
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

All the data template does is to check if the account is locked, and if so it returns the template for a unavailable item, otherwise it returns the data template for an available item.  It also sets the **IsEnabled** propery of the item container to **false** for accounts which are locked so that they cannot be selected.

The `SelectTemplateCore()` method which I override receives 2 parameters.  The first parameter contains data for the GridView item, in my case an Account object.  The second parameter contains the container of the item - in the case of a GridView it is a GridViewItem.  (I casted it to a [SelectorItem](http://msdn.microsoft.com/EN-US/library/windows/apps/windows.ui.xaml.controls.primitives.selectoritem(v=vs.10).aspx) which is the base class of GridViewItem, but you can cast it to a GridViewItem all the same).  The GridViewItem/SelectorItem contains a property IsEnabled which allows you to disable the selection of the item.

Next up we create a XAML resource for the data template selector:

``` xml
<local:AccountDataTemplateSelector x:Key="AccountDataTemplateSelector"
                                    AvailableTemplate="{StaticResource AccountSelectionListViewItemTemplate}"
                                    UnavailableTemplate="{StaticResource AccountSelectionListViewItemTemplateUnavailable}" /></pre>
And finally we specify it as the **DataTemplateSelector** property of the grid view:
<pre class="brush: xml; gutter: true"><GridView Margin="50,120" ItemsSource="{Binding Accounts}"
            SelectionMode="Multiple"
            ItemTemplateSelector="{StaticResource AccountDataTemplateSelector}" 
            />
```

This produce the following output:

![](/assets/images/2013/11/Untitled9.png)

## Changing the item container

One problem which may not be immediately apparent is that the opacity of the entire item container is lowered for items which are disabled.  This may or may not be a problem for you.  In my case I did not like that behaviour as I already overlayed disabled items with the store logo and purchase message and I did not want to opacity of the item lowered as well.  Luckily the solution is quite straightforward and all we need to do is change the item container style and set the **DisabledOpacity** property to 1, so that he opacity of disabled items is not altered:

``` xml
<Style x:Key="AccountSelectionItemContainerStyle" TargetType="GridViewItem">
    <Setter Property="FontFamily" Value="{ThemeResource ContentControlThemeFontFamily}" />
    <Setter Property="FontSize" Value="{ThemeResource ControlContentThemeFontSize}" />
    <Setter Property="Background" Value="Transparent" />
    <Setter Property="TabNavigation" Value="Local" />
    <Setter Property="IsHoldingEnabled" Value="True" />
    <Setter Property="Margin" Value="0,0,2,2" />
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="GridViewItem">
                <GridViewItemPresenter
                    HorizontalContentAlignment="{TemplateBinding HorizontalContentAlignment}"
                    VerticalContentAlignment="{TemplateBinding VerticalContentAlignment}"
                    CheckHintBrush="{ThemeResource ListViewItemCheckHintThemeBrush}"
                    CheckBrush="{ThemeResource ListViewItemCheckThemeBrush}" ContentMargin="4"
                    ContentTransitions="{TemplateBinding ContentTransitions}"
                    CheckSelectingBrush="{ThemeResource ListViewItemCheckSelectingThemeBrush}"
                    DragForeground="{ThemeResource ListViewItemDragForegroundThemeBrush}"
                    DragOpacity="{ThemeResource ListViewItemDragThemeOpacity}"
                    DragBackground="{ThemeResource ListViewItemDragBackgroundThemeBrush}" 
                    DisabledOpacity="1"
                    FocusBorderBrush="{ThemeResource ListViewItemFocusBorderThemeBrush}"
                    Padding="{TemplateBinding Padding}" PointerOverBackgroundMargin="1"
                    PlaceholderBackground="{ThemeResource ListViewItemPlaceholderBackgroundThemeBrush}"
                    PointerOverBackground="{ThemeResource ListViewItemPointerOverBackgroundThemeBrush}"
                    ReorderHintOffset="{ThemeResource ListViewItemReorderHintThemeOffset}"
                    SelectedPointerOverBorderBrush="{ThemeResource ListViewItemSelectedPointerOverBorderThemeBrush}"
                    SelectionCheckMarkVisualEnabled="True"
                    SelectedForeground="{ThemeResource ListViewItemSelectedForegroundThemeBrush}"
                    SelectedPointerOverBackground="{ThemeResource ListViewItemSelectedPointerOverBackgroundThemeBrush}"
                    SelectedBorderThickness="{ThemeResource GridViewItemCompactSelectedBorderThemeThickness}"
                    SelectedBackground="{ThemeResource ListViewItemSelectedBackgroundThemeBrush}" />
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>
```

And specify this new style as the **ItemContainerStyle** of the grid view:

``` xml
<GridView Margin="50,120" ItemsSource="{Binding Accounts}"
            SelectionMode="Multiple"
            ItemTemplateSelector="{StaticResource AccountDataTemplateSelector}" 
            ItemContainerStyle="{StaticResource AccountSelectionItemContainerStyle}"
            />
```

This produces the following output:

![](/assets/images/2013/11/Untitled10.png)

## What about the ListView?

This blog post demonstrated how to get the required behaviour with a WinRT GridView, but you can follow the same process for a WinRT ListView.
