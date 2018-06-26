---
title: Using Tailwind with Angular
description: |
  A look at using the Tailwind CSS framework in Angular applications, allowing you to rapidly build custom user interfaces.
date: 2017-12-20
tags:
- tailwindcss
- angular
url: /blog/using-tailwindcss-with-angular
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

![A Tailwind Card](/assets/images/2017-12-20-using-tailwindcss-with-angular/tailwind-card.png)

For more information on Tailwind you can look at the [documentation](https://tailwindcss.com/docs/). In this blog post I am going to look at how to use Tailwind in a [Angular](https://angular.io/) application, so I will assume you already have some sort of familiarity with both Tailwind and Angular and that you have the [Angular CLI](https://cli.angular.io/) installed.

## Creating the Angular application


Let's use the [Angular CLI](https://cli.angular.io/) to create a new application using SASS:

```bash
ng new angular-tailwind --style=scss
```

Once the application has been created and the NPM packages restored, we can change to the `angular-tailwind` directory containing the application and run it:

```bash
ng serve
```

Browsing to `http://localhost:4200` you will see the familiar screen of a new Angular application:

![Default Angular application](/assets/images/2017-12-20-using-tailwindcss-with-angular/angular-application.png)

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

Next up we will need to update our `/src/styles.scss` file:

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

We will also need to configure the Webpack scripts to process the Tailwind directives properly using [PostCSS](http://postcss.org). In order to do that we will need to [eject the Angular application](https://github.com/angular/angular-cli/wiki/eject):

```bash
 ng eject
 ```

Now, we can update the `webpack.config.js` file that was created. First off load the `tailwindcss` module: 

```js
const tailwindcss = require('tailwindcss');
```

Then find the `postcssPlugins` function and add the line as indicated below:

```js
const postcssPlugins = function () {
        //...
        };
        return [
            postcssUrl([
                {
                    // Only convert root relative URLs, which CSS-Loader won't process into require().
                    // ...
                },
                {
                    // TODO: inline .cur if not supporting IE (use browserslist to check)
                    // ...
                }
            ]),
            tailwindcss('./tailwind-config.js'), // <-- Add this line
            autoprefixer(),
            customProperties({ preserve: true })
        ].concat(minimizeCss ? [cssnano(minimizeOptions)] : []);
    };
```

## Using Tailwind in the application

To use Tailwind, we can add a some HTML markup with the Tailwind styles applied. Let's use the [3D Button sample](https://tailwindcss.com/docs/examples/buttons#3d) from the Tailwind website and update the `/src/app/app.component.html` file as follows:

```html
<div class="text-center mt-8">
  <button class="bg-blue hover:bg-blue-light text-white font-bold py-2 px-4 border-b-4 border-blue-dark hover:border-blue rounded">
    Button
  </button>
</div>
```

Note the use of the Tailwind classes on the `button` element, as well as the use of the `text-center` and `mt-8` classes on the outer `div` element to center the text and add a bit of margin at the top:

![A Tailwind styled button in the Angular application](/assets/images/2017-12-20-using-tailwindcss-with-angular/angular-tailwind-button.png)

## Extracting a component

One of the nice things about Tailwind is that it allows you to [extract components](https://tailwindcss.com/docs/extracting-components). As an example let's extract a `btn-blue` class in order to use that same styling for other buttons in the application.

Head over to the `src/styles.scss` file and update it as follows: 

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

> Note that I removed all those comments from before from the `styles.scss` file.

And now we can use the `btn-blue` style in the application. This time let's add two buttons to `/src/app/app.component.html` using the new style we created:

```html
<div class="text-center mt-8">
  <button class="btn-blue">
    Button 1
  </button>
  <button class="btn-blue">
    Button 2
  </button>
</div>
```

When we run the application, you can see the two button using the `btn-blue` class:

![The button extracted as a Tailwind component](/assets/images/2017-12-20-using-tailwindcss-with-angular/angular-tailwind-component.png)

## Source Code 

Source code for the sample application can be found at [https://github.com/jerriepelser-blog/angular-tailwind](https://github.com/jerriepelser-blog/angular-tailwind)