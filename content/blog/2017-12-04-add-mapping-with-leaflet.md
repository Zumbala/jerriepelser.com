---
title: "Add mapping to your application with Leaflet.js"
description: |
  Leaflet is a great free, open source JavaScript library which allow you to add mapping to your applications.
date: 2017-12-04
tags:
- mapping
- leaflet.js
url: /blog/2017-12-04-add-mapping-with-leaflet
---

When adding mapping to your application, the most common mapping libraries developers seem to use is either [Mapbox](https://www.mapbox.com/) or [Google Maps](https://developers.google.com/maps/). Both of these are great options, but both have limited free tier usage. Once you go past the free tier usage limit or you place the map behind a login, you start paying. 

And the costs can mount up quickly.

There is however a free alternative available in [OpenStreetMap](https://www.openstreetmap.org/). You can use their maps for free as long as you comply with their [license](http://www.openstreetmap.org/copyright). This basically amounts to give credit to the OpenStreetMap contributors in the corner of the map.

![](/assets/images/2017-12-04-add-mapping-with-leaflet/osm-attribution-example.png)

One of the best JavaScript libraries for OpenStreetMap I have come across so far is [Leaflet](http://leafletjs.com/). The folks at Mapbox even [use Leaflet](https://www.mapbox.com/help/define-leaflet/) as the basis for their own Mapbox.js library.

## Get started with Leaflet

To get started with Leaflet, include the CSS file in the `<head>` section of your web page:

```html
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.2.0/dist/leaflet.css"
   integrity="sha512-M2wvCLH6DSRazYeZRIm1JnYyh22purTM+FDB5CsyxtQJYeKq83arPe5wgbNmcFXGqiSH2XR8dT/fJISVA1r/zQ=="
   crossorigin=""/>
```

And include the JavaScript file after the CSS (best somewhere near the bottom of the document before closing the `body` tag):

```html
<body>
...
...
<!-- place near the bottom of the document body -->
<script src="https://unpkg.com/leaflet@1.2.0/dist/leaflet.js"
   integrity="sha512-lInM/apFSqyy1o6s89K4iQUKg6ppXEgsVxT35HbzUupEVRh2Eu9Wdl4tHj7dZO0s1uvplcYGmt3498TtHq+log=="
   crossorigin=""></script>
</body>
```

Now add a `div` element to your document and give it an `id`, e.g. **map**:

```html
 <div id="map"></div>
 ```

 You will also need to give the `div` a height, so add the following CSS to your page:

```css
#map { 
    height: 300px; 
}
```

or if you want to make the map full-screen, you can add the following CSS:

```css
body { margin:0; padding:0; }
#map { position:absolute; top:0; bottom:0; width:100%; }
```

Finally you can create an instance of the map, setting the initial **latitude** to `13.7542`, **longitude** to `100.493` and **zoom level** to `12`. Then, add the OpenStreetMap tile layer to the map, with the proper attribution as per the OpenStreetMap license:

```html
<script>
    var map = L.map('map')
        .setView([13.7542, 100.493], 12);

    L.tileLayer('http://{s}.tile.osm.org/{z}/{x}/{y}.png', {
        attribution: '&copy; <a href="http://osm.org/copyright">OpenStreetMap</a> contributors'
    }).addTo(map);
</script>
```

With that you will have a proper map being displayed:

![](/assets/images/2017-12-04-add-mapping-with-leaflet/leaflet-map.png)

## Other tile sources

You can also use other tile sources to give your map a different style. You can for example go for a more elegant black-and-white map:

```js
var map = L.map('map')
  .setView([13.7542, 100.493], 12);

L.tileLayer('http://{s}.tiles.wmflabs.org/bw-mapnik/{z}/{x}/{y}.png', {
	attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>'
}).addTo(map);
```

![](/assets/images/2017-12-04-add-mapping-with-leaflet/leaflet-map-bw.png)

Or how about a water color one:

```js
var map = L.map('map')
  .setView([13.7542, 100.493], 12);

L.tileLayer('https://stamen-tiles-{s}.a.ssl.fastly.net/watercolor/{z}/{x}/{y}.{ext}', {
	attribution: 'Map tiles by <a href="http://stamen.com">Stamen Design</a>, <a href="http://creativecommons.org/licenses/by/3.0">CC BY 3.0</a> &mdash; Map data &copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>',
	subdomains: 'abcd',
	minZoom: 1,
	maxZoom: 16,
	ext: 'png'
}).addTo(map);
```

![](/assets/images/2017-12-04-add-mapping-with-leaflet/leaflet-map-watercolor.png)

For more information on the available tile providers and samples, you can have a look at the [Leaflet Providers GitHub page](https://github.com/leaflet-extras/leaflet-providers).

## Learn more

To learn more, you can go through their [Tutorials](http://leafletjs.com/examples.html) and [Documentation](http://leafletjs.com/examples.html). There is also a wide [range of plugins available](http://leafletjs.com/plugins.html) to add functionaly such as [Marker Clustering](http://leafletjs.com/plugins.html#clusteringdecluttering), [Heatmaps](http://leafletjs.com/plugins.html#heatmaps), etc.

Leaflet, along with OpenStreetMaps, is definitely a very good, free alternative which you can use instead of the more expensive commercial options like Mapbox, Google Maps and others.

## Source Code 

Not much in terms of external source code to include for this blog post, but [here is the Pen I used](https://codepen.io/jerriep/pen/yPwNJw) for the screenshots in this blog post.