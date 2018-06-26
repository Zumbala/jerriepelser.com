---
title: Using Tailwind with Vue
description: |
  A look at using the Tailwind CSS framework in Vue applications, allowing you to rapidly build custom user interfaces.
date: 2017-12-13
tags:
- tailwindcss
- vuejs
url: /blog/using-tailwindcss-with-vuejs
---

[Tailwind](https://tailwindcss.com/) is a CSS framework which allows you rapidly build custom user interfaces. As opposed to frameworks like Bootstrap, Bulma and Foundation, it is not a UI Kit. It has no theme or components, but instead allow you to construct your own components by combining a bunch of CSS utilities.

So for example, instead of supplying you with a Card component, you can build your own Card by applying the Tailwind CSS styles to a combination of HTML elements.

As an example, here is the markup for a Card component:

```html
<div class="bg-white mx-auto max-w-sm shadow-lg rounded-lg overflow-hidden">
  <div class="sm:flex sm:items-center px-6 py-4">
    <img class="block h-16 sm:h-24 rounded-full mx-auto mb-4 sm:mb-0 sm:mr-4 sm:ml-0" src="https://avatars0.githubusercontent.com/u/1006420?s=460&v=4" alt="">
    <div class="text-center sm:text-left sm:flex-grow">
      <div class="mb-4">
        <p class="text-xl leading-tight">Jerrie Pelser</p>
        <p class="text-sm leading-tight text-grey-dark">Freelance software developer and blogger</p>
      </div>
      <div>
        <button class="text-xs font-semibold rounded-full px-4 py-1 leading-normal bg-white border border-purple text-purple hover:bg-purple hover:text-white">Message</button>
      </div>
    </div>
  </div>
</div>
```

And this is what the Card component will look like:

![A Tailwind Card](/assets/images/2017-12-13-using-tailwindcss-with-vue/tailwind-card.png)

For more information on Tailwind you can look at the [documentation](https://tailwindcss.com/docs/). In this blog post I am going to look at how to use Tailwind in a [Vue](https://vuejs.org/) application, so I will assume you already have some sort of familiarity with both Tailwind and Vue and that you have the [Vue CLI](https://vuejs.org/v2/guide/installation.html#CLI) installed.

## Creating a Vue application

Let's use the [Vue Webpack template](https://github.com/vuejs-templates/webpack) to create our application:

```bash
vue init webpack tailwind-test
```

Once you have created the application, you can change to `tailwind-test` directory containing the application and do an `npm install`:

```bash
npm install
```

And then run the application:

```bash
npm run dev
```

Browsing to `http://localhost:8080` you will see the familiar Vue welcome screen:

![Default Vue application](/assets/images/2017-12-13-using-tailwindcss-with-vue/vue-application.png)

## Installing and configuring Tailwind

To add Tailwind, start off by installing the NPM package:

```bash
npm install tailwindcss --save-dev
```

Next we will need to create a Tailwind configuration file:

```bash
./node_modules/.bin/tailwind init tailwind-config.js
```

For this blog post I will leave the Tailwind configuration file as-is, but you can refer to the [Configuration documentation](https://tailwindcss.com/docs/configuration) for more information on the contents of this file. You can also refer to [Controlling File Size](https://tailwindcss.com/docs/controlling-file-size) for strategies on keeping the generated CSS small.

Next up we will need to create a CSS file. I created the following file at `/src/assets/styles/main.css`:

```css
/**
 * This injects Tailwind's base styles, which is a combination of
 * Normalize.css and some additional base styles.
 *
 * You can see the styles here:
 * https://github.com/tailwindcss/tailwindcss/blob/master/css/preflight.css
 *
 * If using `postcss-import`, you should import this line from it's own file:
 * 
 * @import "./tailwind-preflight.css";
 *
 * See: https://github.com/tailwindcss/tailwindcss/issues/53#issuecomment-341413622
 */
@tailwind preflight;

/**
 * Here you would add any of your custom component classes; stuff that you'd
 * want loaded *before* the utilities so that the utilities could still
 * override them.
 *
 * Example:
 *
 * .btn { ... }
 * .form-input { ... }
 *
 * Or if using a preprocessor or `postcss-import`:
 *
 * @import "components/buttons";
 * @import "components/forms";
 */

/**
 * This injects all of Tailwind's utility classes, generated based on your
 * config file.
 *
 * If using `postcss-import`, you should import this line from it's own file:
 * 
 * @import "./tailwind-utilities.css";
 *
 * See: https://github.com/tailwindcss/tailwindcss/issues/53#issuecomment-341413622
 */
@tailwind utilities;

/**
 * Here you would add any custom utilities you need that don't come out of the
 * box with Tailwind.
 *
 * Example :
 *
 * .bg-pattern-graph-paper { ... }
 * .skew-45 { ... }
 *
 * Or if using a preprocessor or `postcss-import`:
 *
 * @import "utilities/background-patterns";
 * @import "utilities/skew-transforms";
 */
```

> The contents for the file above was taken from the [Tailwind Installation documentation](https://tailwindcss.com/docs/installation).

The final piece of the puzzle is to update the `.postcssrc.js` file which will be in the root folder of your project. Update this file as follows:

```js
module.exports = {
  "plugins": [
    require('postcss-import')(),
    require('tailwindcss')('./tailwind-config.js'),
    require('autoprefixer')(),
  ]
}
```

## Using Tailwind in your application

At this point we have configured Tailwind and we can start using it. Head over to your `App.vue` file and import the CSS file. This is what my `App.vue` looks like:

```html
<template>
  <div id="app">
    <img src="./assets/logo.png">
    <router-view/>
  </div>
</template>

<script>
import '@/assets/styles/main.css'

export default {
  name: 'app'
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
```

And now we can simply add a some HTML markup with the Tailwind styles applied. I will use the [3D Button sample](https://tailwindcss.com/docs/examples/buttons#3d) from the Tailwind website. I updated the `HelloWorld.vue` template as follows:

```html
<template>
  <div class="hello">
    <h1 class="mb-4">{{ msg }}</h1>
    <button class="bg-blue hover:bg-blue-light text-white font-bold py-2 px-4 border-b-4 border-blue-dark hover:border-blue rounded">
      Button
    </button>
  </div>
</template>

<!-- rest of the code for the component is omitted for brevity -->
```

Note the use of the Tailwind classes on the `button` element, as well as the use of the `mb-4` class on the `h1` element to add a bit of bottom margin to the heading.

![A Tailwind styled button in the Vue application](/assets/images/2017-12-13-using-tailwindcss-with-vue/vue-tailwind-button.png)

## Extracting a component

One of the nice things about Tailwind is that it allows you to [extract components](https://tailwindcss.com/docs/extracting-components). As an example I may want to extract a `btn-blue` class in order to use that same styling for other buttons in my application.

Head over to the `main.css` file and update it as follows: 

```css
 @tailwind preflight;

.btn-blue {
  @apply .bg-blue .text-white .font-bold .py-2 .px-4 .border-b-4 .border-blue-dark .rounded
}

.btn-blue:hover {
  @apply .bg-blue-light .border-blue
}

@tailwind utilities;
```

> Note that I removed all those comments from before from the `main.css` file.

And now we can use the `btn-blue` style in our application. This time let's add two buttons to `HelloWorld.vue`:

```html
<template>
  <div class="hello">
    <h1 class="mb-4">{{ msg }}</h1>
    <button class="btn-blue">
      Button 1
    </button>
    <button class="btn-blue">
      Button 2
    </button>
  </div>
</template>


<!-- rest of the code for the component is omitted for brevity -->
```

When we run the application, you can see the two button using the `btn-blue` class:

![The button extracted as a Tailwind component](/assets/images/2017-12-13-using-tailwindcss-with-vue/vue-tailwind-component.png)

## Using Tailwind utilities inside a Vue component

Using Tailwind utilities inside a Vue component is possible, **but definitely not advisable**. The problem is that you will have to inject the Tailwind base styles and utility classes into the `<style>` section for your component:

```html
<template>
  <div class="hello">
    <h1 class="mb-4">{{ msg }}</h1>
    <button class="btn-blue">
      Button 1
    </button>
    <button class="btn-blue">
      Button 1
    </button>
  </div>
</template>

<script>
export default {
  name: 'HelloWorld',
  data () {
    return {
      msg: 'Welcome to Your Vue.js App'
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>

@tailwind preflight;

.btn-blue {
  @apply .bg-blue .text-white .font-bold .py-2 .px-4 .border-b-4 .border-blue-dark .rounded
}

.btn-blue:hover {
  @apply .bg-blue-light .border-blue
}

@tailwind utilities;

</style>
```

This will cause those styles to be repeated for **every component** and your application's CSS **will grow exponentially**. You can read more about it on [this GitHub issue](https://github.com/tailwindcss/tailwindcss/issues/1). The Tailwind team was [considering a way around this](https://github.com/tailwindcss/tailwindcss/pull/169) but at this time have decided not to implement it (yet).

## Source Code

Source code for the sample application can be found at [https://github.com/jerriepelser-blog/vue-tailwind](https://github.com/jerriepelser-blog/vue-tailwind)