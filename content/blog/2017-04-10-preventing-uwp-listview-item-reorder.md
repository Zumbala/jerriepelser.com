---
date: 2017-04-10T00:00:00Z
description: |
  Demonstrates how you can prevent an item in a UWP ListView to be reordered.
tags:
- uwp
- xaml
- listview
title: Preventing a UWP ListView item to be reordered
url: /blog/preventing-uwp-listview-item-reorder/
---

The UWP [ListView](https://docs.microsoft.com/en-us/uwp/api/windows.ui.xaml.controls.listview) allows you to easily reorder items inside a `ListView` by setting the `CanReorderItems` and `AllowDrop` properties to `True`.

Let's for example take a very simple Page with a list view containing 6 buttons. Note that the `CanReorderItems` and `AllowDrop` properties are set to `True`:

```xml
<Page
    x:Class="WorkflowDesigner.MainPage"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:local="using:WorkflowDesigner"
    xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
    mc:Ignorable="d">

    <Grid Background="{ThemeResource ApplicationPageBackgroundThemeBrush}">
        <ListView x:Name="ListView1"
            AllowDrop="True" CanReorderItems="True">
            <Button>Button 1</Button>
            <Button>Button 2</Button>
            <Button>Button 3</Button>
            <Button>Button 4</Button>
            <Button>Button 5</Button>
            <Button>Button 6</Button>
        </ListView>
    </Grid>
</Page>
```

When you run the application, you are able to drag any of the items in the list view to a new position inside the list view:

![](/assets/images/2017-04-10-preventing-uwp-listview-item-reorder/reorder.gif)

But what if you do not want to allow the user to drag and reorder specific items. In the example above, let's say that for some or other reason you do not want to allow the user to drag and reorder the 2nd button. The user can drag and reorder all other buttons around it, but that specific button, we do not want to allow the user to drag.

Alter the list view declaration to set the `CanDragItems` property to `True`, and also add an event handler for the `DragItemsStarting` event.

```xml
<ListView x:Name="ListView1"
    DragItemsStarting="ListView1_OnDragItemsStarting"
    AllowDrop="True" CanReorderItems="True" CanDragItems="True">
    <Button>Button 1</Button>
    <Button>Button 2</Button>
    <Button>Button 3</Button>
    <Button>Button 4</Button>
    <Button>Button 5</Button>
    <Button>Button 6</Button>
</ListView>
```

For the event handler itself, you can check any condition and then simply set the `Cancel` property of the event arguments to `true` if you want to cancel the dragging action. In the example below, I check whether any of the items being dragged is a `Button` with the `Content` **"Button 2"**. If it is, I cancel the drag event, otherwise I allow it.

```csharp
private void ListView1_OnDragItemsStarting(object sender, DragItemsStartingEventArgs e)
{
    e.Cancel = e.Items.Any(o =>
    {
        if (o is Button b && b.Content.ToString() == "Button 2")
            return true;

        return false;
    });
}
```

Now when running the application you can see that I can drag and drop the first button, but when I attempt to drag "Button 2", the drag operation will simply not be initiated:

![](/assets/images/2017-04-10-preventing-uwp-listview-item-reorder/reorder-cancel.gif)

BTW, if you are not familiar with the syntax `o is Button b` which I used above, then please check out the **Pattern Matching** section of the [What’s New in C# 7.0](https://blogs.msdn.microsoft.com/dotnet/2016/08/24/whats-new-in-csharp-7-0/) blog post, or the Roslyn feature document on [Pattern Matching for C#](https://github.com/dotnet/roslyn/blob/features/patterns/docs/features/patterns.md).