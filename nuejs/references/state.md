# Nuestate - URL-first state management, complete reference

Application state lives in the URL by default. Bookmarking, sharing, and back/forward navigation work with no extra code. No stores, no reducers, no actions: read and write properties on a single proxy object. The library is under 2KB with zero dependencies.

## The state proxy

```js
import { state } from 'state'
```

Nuekit maps `state` to `/@nue/state.js` via import maps automatically; no path needed. State is populated from the current URL and browser storage when the application starts.

```js
console.log(state.view)    // current value or undefined
state.view = 'dashboard'   // writing updates URL/storage and notifies listeners
```

## state.setup(config)

Configure where each piece of state is stored and how routing works. Call once, early:

```js
state.setup({
  route: '/shop/:category/:product',          // URL path parameters
  query: ['search', 'color', 'size', 'page'], // URL query string (?search=...)
  session: ['user', 'cart'],                  // sessionStorage (survives reload, not new tab)
  local: ['theme', 'currency'],               // localStorage (persists across sessions)
  memory: ['loading', 'errors', 'removeId'],  // in-memory only, lost on reload
  emit_only: ['deleted', 'saved'],            // event signals, never stored
  autolink: true                              // intercept matching <a href> links for SPA navigation
})
```

Storage type selection: route for shareable navigation state, query for shareable filters, session for per-tab user context, local for durable preferences, memory for transient UI flags, emit_only for fire-and-forget events.

Writing a route or query property updates the URL:

```js
state.setup({ route: '/app/:section/:id' })

state.section = 'products'   // URL: /app/products
state.id = '123'             // URL: /app/products/123
state.search = 'shoes'       // URL: /app/products/123?search=shoes (if in query list)
```

With `autolink: true`, clicks on links matching the route pattern update state instead of triggering a page load. Regular `<a href="/app/products/123">` markup becomes SPA navigation; no router component exists or is needed.

## state.on(properties, callback)

Listen to changes. The first argument is a space-separated property list; the callback receives an object containing only the changed properties:

```js
state.on('search filter page', (changes) => {
  console.log('Changed:', changes)
})

// async data fetching driven by state
state.on('search filter', async (changes) => {
  const results = await fetchResults(changes.search, changes.filter)
  state.results = results
})
```

## state.init()

Populates state from the current URL on first load. Call it in the root component's `mounted()` so listeners fire for the initial route.

## SPA root pattern

The combination of `<!doctype dhtml>` and a `<body>` scope tells Nue the file handles all routes in its directory:

```html
<!doctype dhtml>

<script>
  import { state } from 'state'

  state.setup({ route: '/:id', autolink: true })
</script>

<body>
  <main>
    <article/>
  </main>

  <script>
    // swap the mounted component when the URL changes
    state.on('id', ({ id }) => {
      this.mount(id ? 'user' : 'users', 'article')
    })

    // initialize from the current URL
    mounted() {
      state.init()
    }
  </script>
</body>
```

How this works: `/:id` captures URLs like `/123` or `/alice`, setting `state.id` to the captured value. The `state.on('id')` listener decides which component to mount into the `<article/>` placeholder. `autolink` turns plain anchors into client-side navigation.

## Using state inside components

State reads interpolate like any expression, and DOM events write back:

```html
<input value="{ state.search }" :oninput="state.search = $event.target.value">
```

Do not duplicate state into component-local copies when the value belongs in the URL. Component-local `this.prop` is for purely internal, non-shareable view state.
