---
date: 2013-10-21T00:00:00Z
description: |
  Discusses 3 different options you can use to add modern icons to your Windows 8.1 XAML application.
tags:
- design
- icons
- winrt
- xaml
title: 3 Ways to add modern icon goodness to your Windows 8.1 app
url: /blog/3-ways-to-add-modern-icon-goodness-windows-8-1/
---

So you're building an app for Windows 8.1 and you are looking for some nice icons.  There are quite a number of sources for XAML icons which not only works for Windows 8.1, but also for Windows Phone 8 and other XAML platforms.

## 1. Segoe UI Symbol font

Using the Segoe UI Symbol font is probably the easiest way to get icons that works with Windows 8.1 without too much fuss.  For more information on the available icons available in the font, check out this [nice list on the MSDN library](http://msdn.microsoft.com/en-us/library/windows/apps/jj841126.aspx).

To use them could not be any easier.  The following XAML snippet displays the camera icon:

``` csharp
<TextBlock FontFamily="Segoe UI Symbol" FontSize="40">&#xE114;</TextBlock>
```

Assuming you want to display the icon in an appbar, you can use the following XAML markup:

``` xml
<AppBarButton Label="Font Icon">
    <AppBarButton.Icon>
        <FontIcon FontFamily="Segoe UI Symbol" Glyph="&#xE114;"/>
    </AppBarButton.Icon>
</AppBarButton>
```

Actually, since you are using an icon from the standard Segoe UI Symbol font, this can be even further simplified by using a symbol icon:

``` xml
<AppBarButton Label="Symbol Icon" Icon="Camera"/>
```

For more information check out the samples in the [documentation for the AppBarButton class](http://msdn.microsoft.com/en-us/library/windows/apps/windows.ui.xaml.controls.appbarbutton.aspx).

**Please note**: You are not limited to using AppBarButton only in the AppBar.  You can for example replace your normal buttons in your XAML with AppBar buttons

## 2. Other XAML Icons

You are not only limited to the icons in the Segoe UI Symbol font for the icons in your app bar.  Looking at the documentation for the AppBar class you would have noticed that you can also specify a path icon, and there are plenty of sources available to find path-based XAML icons.  Here are a few of them:

*   Modern UI Icons ([http://modernuiicons.com](http://modernuiicons.com))
*   The XAML Project ([http://www.thexamlproject.com](http://www.thexamlproject.com))
*   Metro Studio ([http://www.syncfusion.com/downloads/metrostudio](http://www.syncfusion.com/downloads/metrostudio))
Let us say for example that we want to use a camera icon from the Modern UI Icons website.  Simply surf on over to [http://modernuiicons.com](http://modernuiicons.com) and search for "camera".  Click on the camera icon you like and you will be presented with XAML markup to display the icon, as in the screenshot below

![](/assets/images/2013/10/Capture2.png)

That is however much more markup than what we need.  All we are interested is the path data, so simply highlight and copy the Data property of the Path and use that in a path icon:

``` xml
<AppBarButton Label="Path Icon" HorizontalAlignment="Stretch" VerticalAlignment="Stretch">
    <AppBarButton.Icon>
        <PathIcon Data="F1 M 30,27C 30,24.3766 32.3767,22 35,22L 41,22C 43.6234,22 46,24.3766 46,27L 50.9999,27.0001C 53.7613,27.0001 55.9999,29.2387 55.9999,32.0001L 55.9999,46.0001C 55.9999,48.7615 53.7613,51.0001 50.9999,51.0001L 25,51.0001C 22.2385,51.0001 20,48.7615 20,46.0001L 20,32.0001C 20,29.2387 22.2385,27.0001 25,27.0001L 30,27 Z M 25.5,30C 24.6716,30 24,30.8954 24,32C 24,33.1046 24.6716,34 25.5,34C 26.3284,34 27,33.1046 27,32C 27,30.8954 26.3284,30 25.5,30 Z M 38,32C 34.134,32 31,35.134 31,39C 31,42.866 34.134,46 38,46C 41.866,46 45,42.866 45,39C 45,35.134 41.866,32 38,32 Z M 38,34.5C 40.4853,34.5 42.5,36.5147 42.5,39C 42.5,41.4853 40.4853,43.5 38,43.5C 35.5147,43.5 33.5,41.4853 33.5,39C 33.5,36.5147 35.5147,34.5 38,34.5 Z"/>
    </AppBarButton.Icon>
</AppBarButton>
```

The problem with the above example however is that the icons on [http://modernuiicons.com/](http://modernuiicons.com/) are designed for a dimension of 76x76 pixels, and the PathIcon class expects a dimension of 40x40 pixels.  The path therefore needs to be scaled to fit in the correct size.

You will notice below the icon in the left is the one which I used as-is and is not displaying correctly in the expected dimensions.  The one on the right was scaled to display correctly.

![](/assets/images/2013/10/Untitled5.png)

### Scaling paths to the required dimensions

As mentioned above, using vector icons from other sources can be problematic as the dimensions may be different from what is expected.  When I first tried to use a path icon with a path from another source I was surprised to see that the path did not scale to automatically fit to the dimensions of the AppBarButton.  [I raised the issue in the Windows 8.1 developer forums](http://social.msdn.microsoft.com/Forums/windowsapps/en-US/c535bd33-90f0-428c-8b05-db6fe27e9644/scaling-of-pathicon-in-appbarbutton?forum=w81prevwCsharp) and was promptly supplied with a very nice explanation and solution to the problem by Thomas Huber.   His solution was based on using the the [WPF Geometry Transformation Tool](http://wpftutorial.net/GeometryTransformer.html) to scale the path down to the required dimensions.

This involves knowing what the dimensions of the original path is and applying a little bit of math to understand by what factor it should be scaled to get to the required dimensions of 40 x 40 pixels.  So in my case I simply divide 40 (target dimension) by 76 (source dimension) which leaves me with a scaling factor of **0.5263157894736842**.  I open the WPF Geometry Transformer tool, paste the original path in the **Input Geometry** field, supply the values for the scale and click the **Transform** button.  The tool will display the transformed path in the **Output Geometry** field, so copy that and use it as the data for your path icon.

![](/assets/images/2013/10/Capture3.png)

## 3. Other Sources of Flatness

If the above sources does not fulfil your craving for icons you are in luck because with iOS 7 the whole world has gone <del>nuts</del> flat and there are plenty of goodies out there.  If you don't know how to use these in your XAML projects, do not fear for I have created a short screencast below which shows you how to do it.

Here are a few links with sources for flat icons:

**Mashable** has a nice collection of flat design icons, which should be more than enough to supply you with your needs.  Check it out at [http://mashable.com/2013/08/14/flat-design-icons](http://mashable.com/2013/08/14/flat-design-icons/)

[Graphic River](http://graphicriver.net/) (one of the [Envato](http://www.envato.com/) websites) has a lot of graphic design assets.  Go check out their [icons category](http://graphicriver.net/popular_item/by_category?category=icons), it is filled to the brim with flatness.

If the above does not help, just **Google** "flat ui icons".  Here, [let me do that for you](http://lmgtfy.com/?q=flat+ui+icons).

So here is that screencast which I promised that shows you how to find a generic vector based icon and use it in a Windows 8.1 project.

<iframe width="640" height="480" src="//www.youtube.com/embed/g2o23mDYPKE" frameborder="0" allowfullscreen></iframe>


## Conclusion

So as you can see there is no shortage of flat icons on the web.  One of the problems which I have noticed over the years was that developers would get and use icons from a variety of different sources in their application.  This would be fairly evident as it was obvious that the icons used different visual styles.  With flat icons there can still be visual inconsistency with icons which uses a different stroke thickness for example.  It should however be far easier to get really nice looking icons which are visually consistent with the rest of your user interface.  I hope that this blog post gave you a few ideas of where to find extra icons for use in your Windows 8.1 application.

If you have anything to add, please leave feedback in the comments section below.