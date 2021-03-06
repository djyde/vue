# vue-server-renderer

> This package is auto-generated. For pull requests please see [src/entries/web-server-renderer.js](https://github.com/vuejs/vue/blob/next/src/entries/web-server-renderer.js).

This package offers Node.js server-side rendering for Vue 2.0.

## Installation

``` bash
npm install vue-server-renderer
```

## API

### createRenderer([[rendererOptions](#renderer-options)])

Create a `renderer` instance.

``` js
const renderer = require('vue-server-renderer').createRenderer()
```

---

### renderer.renderToString(vm, cb)

Render a Vue instance to string. The callback is a standard Node.js callback that receives the error as the first argument:

``` js
const Vue = require('vue')

const renderer = require('vue-server-renderer').createRenderer()

const vm = new Vue({
  render (h) {
    return h('div', 'hello')
  }
})

renderer.renderToString(vm, (err, html) => {
  console.log(html) // -> <div server-rendered="true">hello</div>
})
```

---

### renderer.renderToStream(vm)

Render a Vue instance in streaming mode. Returns a Node.js readable stream.

``` js
// example usage with express
app.get('/', (req, res) => {
  const vm = new App({ url: req.url })
  const stream = renderer.renderToStream(vm)

  res.write(`<!DOCTYPE html><html><head><title>...</title></head><body>`)

  stream.on('data', chunk => {
    res.write(chunk)
  })

  stream.on('end', () => {
    res.end('</body></html>')
  })
})
```

---

### createBundleRenderer(code, [[rendererOptions](#renderer-options)])

Creates a `bundleRenderer` instance using pre-bundled application code (see [Creating the Server Bundle](#creating-the-server-bundle)). For each render call, the code will be re-run in a new context using Node.js' `vm` module. This ensures your application state is discrete between requests, and you don't need to worry about structuring your application in a limiting pattern just for the sake of SSR.

``` js
const bundleRenderer = require('vue-server-renderer').createBundleRenderer(code)
```

---

### bundleRenderer.renderToString([context], cb)

Render the bundled app to a string. Same callback interface with `renderer.renderToString`. The optional context object will be passed to the bundle's exported function.

``` js
bundleRenderer.renderToString({ url: '/' }, (err, html) => {
  // ...
})
```

---

### bundleRenderer.renderToStream([context])

Render the bundled app to a stream. Same stream interface with `renderer.renderToStream`. The optional context object will be passed to the bundle's exported function.

``` js
bundleRenderer
  .renderToStream({ url: '/' })
  .pipe(writableStream)
```

## Creating the Server Bundle

The application bundle can be generated by any build tool, so you can easily use Webpack + `vue-loader` with the bundleRenderer. You do need to use a slightly different webpack config and entry point for your server-side bundle, but the difference is rather minimal:

1. add `target: 'node'`, and use `output: { libraryTarget: 'commonjs2' }` for your webpack config.

2. In your server-side entry point, export a function. The function will receive the render context object (passed to `bundleRenderer.renderToString` or `bundleRenderer.renderToStream`), and should return a Promise, which should eventually resolve to the app's root Vue instance:

  ``` js
  // server-entry.js
  import Vue from 'vue'
  import App from './App.vue'

  const app = new Vue(App)

  // the default export should be a function
  // which will receive the context of the render call
  export default context => {
    // data pre-fetching
    return app.fetchServerData(context.url).then(() => {
      return app
    })
  }
  ```

## Renderer Options

### directives

Allows you to provide server-side implementations for your custom directives:

``` js
const renderer = createRenderer({
  directives: {
    example (vnode, directiveMeta) {
      // transform vnode based on directive binding metadata
    }
  }
})
```

As an example, check out [`v-show`'s server-side implementation](https://github.com/vuejs/vue/blob/next/src/platforms/web/server/directives/show.js).

---

### cache

> Note: this option has changed and is different from versions <= 2.0.0-alpha.8.

Provide a cache implementation. The cache object must implement the following interface:

``` js
{
  get: (key: string, [cb: Function]) => string | void,
  set: (key: string, val: string) => void,
  has?: (key: string, [cb: Function]) => boolean | void // optional
}
```

A typical usage is passing in an [lru-cache](https://github.com/isaacs/node-lru-cache):

``` js
const LRU = require('lru-cache')

const renderer = createRenderer({
  cache: LRU({
    max: 10000
  })
})
```

Note that the cache object should at least implement `get` and `set`. In addition, `get` and `has` can be optionally async if they accept a second argument as callback. This allows the cache to make use of async APIs, e.g. a redis client:

``` js
const renderer = createRenderer({
  cache: {
    get: (key, cb) => {
      redisClient.get(key, (err, res) => {
        // handle error if any
        cb(res)
      })
    },
    set: (key, val) => {
      redisClient.set(key, val)
    }
  }
})
```

## Component-Level Caching

You can easily cache components during SSR by implementing the `serverCacheKey` function:

``` js
export default {
  props: ['item'],
  serverCacheKey: props => props.item.id,
  render (h) {
    return h('div', this.item.id)
  }
}
```

The cache key is per-component, and it should contain sufficient information to represent the shape of the render result. The above is a good implementation because the render result is solely determined by `props.item.id`. However, if the render result also relies on another prop, then you need to modify your `getCacheKey` implementation to take that other prop into account.

Returning a constant will cause the component to always be cached, which is good for purely static components.

If the renderer hits a cache for a component during render, it will directly reuse the cached result for the entire sub tree. So **do not cache a component containing child components that rely on global state**.

In most cases, you shouldn't and don't need to cache single-instance components. The most common type of components that need caching are ones in big lists. Since these components are usually driven by objects in database collections, they can make use of a simple caching strategy: generate their cache keys using their unique id plus the last updated timestamp:

``` js
serverCacheKey: props => props.item.id + '::' + props.item.last_updated
```

## Externals

By default, we will bundle every dependency of our app into the server bundle as well. V8 is very good at optimizing running the same code over and over again, so in most cases the cost of re-running it on every request is a worthwhile tradeoff in return for more freedom in application structure.

You can also further optimize the re-run cost by externalizing dependencies from your bundle. When running the bundle, any raw `require()` calls found in the bundle will return the actual module from your rendering process. With Webpack, you can simply list the modules you want to externalize using the `externals` config option. This avoids having to re-initialize the same module on each request and can also be beneficial for memory usage.

However, since the same module instance will be shared across every request, you need to make sure that the dependency is **idempotent**. That is, using it across different requests should always yield the same result - it cannot have global state that may be changed by your application. Because of this, you should avoid externalizing Vue itself and its plugins.

## Client Side Hydration

In server-rendered output, the root element will have the `server-rendered="true"` attribute. On the client, when you mount a Vue instance to an element with this attribute, it will attempt to "hydrate" the existing DOM instead of creating new DOM nodes.

In development mode, Vue will assert the client-side generated virtual DOM tree matches the DOM structure rendered from the server. If there is a mismatch, it will bail hydration, discard existing DOM and render from scratch. **In production mode, this assertion is disabled for maximum performance.**

### Hydration Caveats

One thing to be aware of when using SSR + client hydration is some special HTML structures that may be altered by the browser. For example, when you write this in a Vue template:

``` html
<table>
  <tr><td>hi</td></tr>
</table>
```

The browser will automatically inject `<tbody>` inside `<table>`, however, the virtual DOM generated by Vue does not contain `<tbody>`, so it will cause a mismatch. To ensure correct matching, make sure to write valid HTML in your templates.
