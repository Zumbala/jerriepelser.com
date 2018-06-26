---
title: "TIL: The @ symbol in JavaScript import statement"
description: |
  A quick explainer of the @ symbol in the JavaScript imports statements for Vue applications.
date: 2017-12-11
tags:
- javascript
- vuejs
- webpack
url: /blog/til-at-symbol-javascript-import
---

_Today I Learned (TIL): About the usage of the @ symbol in the JavaScript import statement._

I am busy working on a new project and I am using [Vue](https://vuejs.org/) to develop the front-end. I used the [Vue Webpack template](https://github.com/vuejs-templates/webpack) to bootstrap the project, and I have noticed the usage of the `@` symbol in the `import` statement in various places. For example, here is the imports from my `router/index.js` file:

```js
import Vue from 'vue'
import Router from 'vue-router'
import Home from '@/components/Home'
import AuthCallback from '@/components/AuthCallback'
import Map from '@/components/Map'


//...
```

Not sure about what this means, I ~~Googled~~ DuckDuckGo'd this and came across the [following StackOverflow answer](https://stackoverflow.com/questions/42749973/es6-import-using-at-sign-in-path-in-a-vue-js-project-using-webpack).

It turns out that it comes from the following section of the `build/webpack.base.conf.js` file:

```js
resolve: {
    extensions: ['.js', '.vue', '.json'],
    alias: {
      'vue$': 'vue/dist/vue.esm.js',
      '@': resolve('src'),
    }
  },
```

In other words, this is part of the [Webpack configuration](https://webpack.github.io/docs/configuration.html) and specifically the [resolve.alias](https://webpack.github.io/docs/configuration.html#resolve-alias) option which allow you to _replace modules with other modules or paths_.

The code for the `resolve` function looks as follows which is being called when Webpack comes across an `@` symbol looks as follows:

```js
function resolve (dir) {
  return path.join(__dirname, '..', dir)
}
```

To understand what is happening, let's assume the root folder for my project is `/development/slackmap/frontend` and this is what the folder structure of my project looks like:

![](/assets/images/2017-12-11-til-at-symbol-javascript-import/folder-structure.png)

So when Webpack comes across this:

```js
import Home from '@/components/Home'
```

It will call the `resolve` function and replace the `@` symbol with the result of that. So it will effectively replace the `@` symbol with  `/development/slackmap/frontend/build/../src` which means the path to the module becomes `/development/slackmap/frontend/build/../src/components/Home` or more specifically `/development/slackmap/frontend/src/components/Home`. So it is a handy way of not having to use relative paths, but instead being able to use an absolute path from the root folder of the project.

As someone who is not too clued up on JavaScript (but busy learning) the following comment from one of the people on the linked StackOverflow answer appeals a lot to me:

> JavaScript just isn't JavaScript anymore. Babel/webpack gives us this Frankenstein language and somehow new developers are meant to know where ECMAScript spec ends and userland plugins/transforms begin. It's really sad, imo. 

I find a lot of this new and a bit confusing, but building something and asking questions is the only way of learning. So forward I go.

Hope you find this little tip useful. I sure did 😀