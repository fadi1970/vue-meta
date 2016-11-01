<p align="center">
  <img src="http://imgh.us/Group_5.svg" alt="vue-meta">
</p>

<h5 align="center">
  Manage page meta info in Vue components. SSR + Streaming supported.
</h5>

<p align="center">
  <a href="https://github.com/feross/standard">
    <img src="https://cdn.rawgit.com/feross/standard/master/badge.svg" alt="Standard - JavaScript Style">
  </a>
</p>

<p align="center">
<a href="https://github.com/declandewet/vue-meta/releases/latest"><img src="https://img.shields.io/github/release/declandewet/vue-meta.svg" alt="github release"></a> <a href="http://npmjs.org/package/vue-meta"><img src="https://img.shields.io/npm/v/vue-meta.svg" alt="npm version"></a> <a href="https://travis-ci.org/declandewet/vue-meta"><img src="https://travis-ci.org/declandewet/vue-meta.svg?branch=master" alt="Build Status"></a> <a href="https://codecov.io/gh/declandewet/vue-meta"><img src="https://codecov.io/gh/declandewet/vue-meta/branch/master/graph/badge.svg" alt="codecov"></a><br>
<a href="https://david-dm.org/declandewet/vue-meta"><img src="https://david-dm.org/declandewet/vue-meta/status.svg" alt="dependencies Status"></a> <a href="https://david-dm.org/declandewet/vue-meta?type=dev"><img src="https://david-dm.org/declandewet/vue-meta/dev-status.svg" alt="devDependencies Status"></a><br>
<a href="http://npm-stat.com/charts.html?package=vue-meta"><img src="https://img.shields.io/npm/dm/vue-meta.svg" alt="npm downloads"></a> <a href="https://gitter.im/vue-meta/Lobby"><img src="https://badges.gitter.im/vue-meta/Lobby.svg" alt="Gitter"></a>
</p>

<br>
<hr style="height: 1px">
<br>

> **Please note** that this project is still in very early alpha development and is *not* considered to be production ready.
> **You have been warned.** There is no sanitization yet, no tests, and you might even find some features are still missing.

```html
<template>
  ...
</template>

<script>
  export default {
    metaInfo: {
      title: 'My Example App', // set a title
      titleTemplate: '%s - Yay!', // title is now "My Example App - Yay!"
      htmlAttrs: {
        lang: 'en',
        amp: undefined // "amp" has no value
      }
    }
  }
</script>
```

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
# Table of Contents

- [Description](#description)
- [Install](#install)
- [Usage](#usage)
    - [Step 1: Preparing the plugin](#step-1-preparing-the-plugin)
    - [Step 2: Exposing `$meta` to `bundleRenderer`](#step-2-exposing-meta-to-bundlerenderer)
    - [Step 3: Server-side rendering with `inject()`](#step-3-server-side-rendering-with-inject)
      - [`renderToString()`](#rendertostring)
      - [`renderToStream()`](#rendertostream)
    - [Step 4: Start defining `metaInfo`](#step-4-start-defining-metainfo)
- [FAQ](#faq)
  - [How do I use component data in `metaInfo`?](#how-do-i-use-component-data-in-metainfo)
  - [How do I use component props in `metaInfo`?](#how-do-i-use-component-props-in-metainfo)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Description
`vue-meta` is a [Vue 2.0](https://vuejs.org) plugin that allows you to manage your app's meta information, much like [`react-helmet`](https://github.com/nfl/react-helmet) does for React. However, instead of setting your data as props passed to a proprietary component, you simply export it as part of your component's data using the `metaInfo` property.

These properties, when set on a deeply nested component, will cleverly overwrite their parent components' `metaInfo`, thereby enabling custom info for each top-level view as well as coupling meta info directly to deeply nested subcomponents for more maintainable code.

# Install 

```sh
$ yarn add vue-meta
# or $ npm install vue-meta --save
```

# Usage

### Step 1: Preparing the plugin
> This step is optional if you don't need SSR and `Vue` is available as a global variable. `vue-meta` will install itself in this case.

In order to use this plugin, you first need to pass it to `Vue.use` in a file that runs both on the server and on the client before your root instance is mounted. If you're using [`vue-router`](https://github.com/vuejs/vue-router), then your main `router.js` file is a good place:

**router.js:**
```js
import Vue from 'vue'
import Router from 'vue-router'
import Meta from 'vue-meta'

Vue.use(Router)
Vue.use(Meta)

export default new Router({
  ...
})
```

If you don't care about server-side rendering, you can skip straight to [step 4](#step-4-start-defining-metainfo). Otherwise, continue. :smile:

### Step 2: Exposing `$meta` to `bundleRenderer`

You'll need to expose the results of the `$meta` method that `vue-meta` adds to the Vue instance to the bundle render context before you can begin injecting your meta information. You'll need to do this in your server entry file:

**server-entry.js:**
```js
import app from './app'

const router = app.$router
const store = app.$store
const meta = app.$meta() // here

export default (context) => {
  router.push(context.url)
  return Promise.all(
    router.getMatchedComponents().map(
      (component) => component.preFetch
        ? component.preFetch(store)
        : component
    )
  )
    .then(() => {
      context.initialState = store.state
      context.meta = meta // and here
      return app
    })
}
```

### Step 3: Server-side rendering with `inject()`

All that's left for you to do now before you can begin using `metaInfo` options in your components is to make sure they work on the server by `inject`-ing them. You have two methods at your disposal:

#### `renderToString()`

Considerably the easiest method to wrap your head around is if your Vue server markup is rendered out as a string:

**server.js:**

```js
...
app.get('*', (request, response) => {
  const context = { url: request.url }
  
  renderer.renderToString(context, (error, html) => {
    if (error) {
      ...
    } else {
      const { initialState, meta } = context
      const metaInfo = meta.inject()
      
      response.send(`
        <!doctype html>
        <html ${metaInfo.htmlAttrs.toString()}>
          <head>
            ${metaInfo.title.toString()}
            <script>
              window.__INITIAL_STATE__ = ${ !initialState
                ? '{}'
                : serialize(initialState, { isJSON: true })
              }
            </script>
          </head>
          <body>
            ${html}
            <script src="/assets/vendor.bundle.js"></script>
            <script src="/assets/client.bundle.js"></script>
          </body>
        </html>
      `)
    }
  })
})
...
```

#### `renderToStream()`

A little more complex, but well worth it, is to instead stream your response. `vue-meta` supports streaming with no effort (on it's part :stuck_out_tongue_winking_eye:) thanks to Vue's clever `bundleRenderer` context injection:

**server.js**
```js
app.get('*', (request, response) => {
  const context = { url: request.url }
  const renderStream = renderer.renderToStream(context)
  let firstChunk = true
  
  response.write('<!doctype html>')
  
  renderStream.on('data', (chunk) => {
    if (firstChunk) {
      const metaInfo = context.meta.inject()
      
      if (metaInfo.htmlAttrs) {
        response.write(`<html ${metaInfo.htmlAttrs.toString()}>`)
      }
      
      response.write('<head>')
      
      if (metaInfo.title) {
        response.write(metaInfo.title.toString())
      }
      
      response.write('</head><body>')
      
      if (context.initialState) {
        response.write(
          `<script>window.__INITIAL_STATE__=${
            serialize(context.initialState, { isJSON: true })
          }</script>`
        )
      }
      
      firstChunk = false
    }
    response.write(chunk)
  })
  
  renderStream.on('end', () => {
    response.end(`
      <script src="/assets/vendor.bundle.js"></script>
      <script src="/assets/client.bundle.js"></script>
      </body></html>
    `)
  })
  
  renderStream.on('error', (error) => {
    response.status(500).end(`<pre>${error.stack}</pre>`)
  })
})
```

### Step 4: Start defining `metaInfo`

In any of your components, define a `metaInfo` property:

**App.vue:**
```html
<template>
  <div id="app">
    <router-view></router-view>
  </div>
</template>

<script>
  export default {
    name: 'App',
    metaInfo: {
      // if no subcomponents specify a metaInfo.title, this title will be used
      title: 'Default Title',
      // all titles will be injected into this template
      titleTemplate: '%s | My Awesome Webapp'
    }
  }
</script>
```

**Home.vue**
```html
<template>
  <div id="page">
    <h1>Home Page</h1>
  </div>
</template>

<script>
  export default {
    name: 'Home',
    metaInfo: {
      title: 'My Awesome Webapp',
      // override the parent template and just use the above title only
      titleTemplate: null
    }
  }
</script>
```

**About.vue**
```html
<template>
  <div id="page">
    <h1>About Page</h1>
  </div>
</template>

<script>
  export default {
    name: 'About',
    metaInfo: {
      // title will be injected into parent titleTemplate
      title: 'About Us'
    }
  }
</script>
```

# FAQ

Here are some answers to some frequently asked questions.

## How do I use component data in `metaInfo`?
Specify a function instead of a property.

**BlogPost.vue:**
```html
<template>
  <div id="page">
    <h1>{{ title }}</h1>
  </div>
</template>

<script>
  export default {
    name: 'BlogPost',
    data: () => ({
      title: 'Sample blog post'
    }),
    metaInfo: {
      title () {
        return this.title
      }
    }
  }
</script>
```

## How do I use component props in `metaInfo`?
The same way you use data.

**BlogPostWrapper.vue**
```html
<template>
  <div id="page">
    <blog-post :title="title"></blog-post>
  </div>
</template>

<script>
  import BlogPost from './BlogPost.vue'
  
  export default {
    name: 'BlogPostWrapper',
    components: { BlogPost },
    data: () => ({
      title: 'Example blog post'
    })
  }
</script>
```

**BlogPost.vue**
```html
<template>
  <div id="page">
    <h1>{{ title }}</h1>
  </div>
</template>

<script>
  export default {
    name: 'BlogPost',
    props: ['title'],
    metaInfo: {
      title () {
        return this.title
      }
    }
  }
</script>
```
