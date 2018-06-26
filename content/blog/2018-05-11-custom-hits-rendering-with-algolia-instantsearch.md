---
title: "Custom Hits rendering with Algolia InstantSearch"
description:
  Demonstrates how you can customise the rendering of the result hits when using the Hits widget in Algolia InstantSearch
tags:
- algolia
- instantsearch
- search
---

[Algolia InstantSearch for JavaScript](https://community.algolia.com/instantsearch.js/) makes uses of [Widgets](https://community.algolia.com/instantsearch.js/v2/widgets.html) to control the rendering of various parts of your search User Interface. This process is nicely demonstrated in their [Getting Started](https://community.algolia.com/instantsearch.js/v2/getting-started.html) guide.

One of the widgets I use is the [Hits widget](https://community.algolia.com/instantsearch.js/v2/widgets/hits.html) to display the list of results from the current search. I make use of the `item` template to control the rendering of these results.

## Basic search result rendering

This is the simplfied source for my current configuration:

```html
<div id="hits">
    <!-- Hits widget will appear here -->
</div>

<script>
    const search = instantsearch({
        appId: '...',
        apiKey: '...',
        indexName: 'jobs',
        routing: true
    });

    search.addWidget(
        instantsearch.widgets.hits({
            container: '#hits',
            templates: {
                empty: 'No results',
                item: '<p>{{ title }}</p>'
            }
        })
    );
</script>
```

Which will produce the following HTML output:

```html
<div id="hits">
    <div class="ais-hits">
        <div class="ais-hits--item">
            <p>.NET Developer</p>
        </div>
        <div class="ais-hits--item">
            <p>.NET Team Lead</p>
        </div>
        <div class="ais-hits--item">
            <p>Senior .Net Developer</p>
        </div>
        ...
    </div>
</div>
```

## Custom search result rendering

The HTML template I am using provides nice styling for rendering search results, but this requires me to use a `<ul>` element with child `<li>` elements. The problem, however, is that I have no control of the outer elements which are rendered in the search results. In the output above, this is the outer `<div class="ais-hits">` element as well as the `<div class="ais-hits--item">` which surrounds every individual result item.

Using the `item` template of the Hits widget, I can only control the inner content of each search result item. In my case, this is the `<p>...</p>` element in the output above.

It turns out that the Hits widget also has a `allItems` template which you can use for rendering the list of all the results. You use this in place of the `item` template, as the two cannot be used together.

The `allItems` template makes the list of results available in a `hits` key that you can use in your [Mustache](https://mustache.github.io/mustache.5.html) template. So with that in mind, here is the simplified version of my source code when using the `allItems` template to control rendering of the search results:

```html
<div id="hits">
    <!-- Hits widget will appear here -->
</div>

<script>
    const search = instantsearch({
        appId: '...',
        apiKey: '...',
        indexName: 'jobs',
        routing: true
    });

    search.addWidget(
        instantsearch.widgets.hits({
            container: '#hits',
            templates: {
                empty: 'No results',
                allItems: `<ul>
                                {{#hits}}
                                <li>
                                    {{ title }}
                                </li>
                                {{/hits}}
                            </ul>`
            }
        })
    );
</script>
```

This produces the following HTML output:

```html
<div id="hits">
    <div class="ais-hits">
        <ul>
            <li>
                .NET Developer
            </li>
            <li>
                .NET Team Lead
            </li>
            <li>
                Senior .Net Developer
            </li>
            ...
        </ul>
    </div>
</div>
```
