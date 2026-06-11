# Nueserver - edge-first backend, complete reference

Server code lives in `@shared/server/` (typically `index.js` or `index.ts`). The development environment uses edge-compatible patterns from day one, so the same code runs identically on your machine and on Cloudflare edge locations.

## Global route handlers

No imports, no app instance, no server setup, no Hono router, no Express. The functions `get()`, `post()`, and `use()` are globals:

```js
get('/api/users', async (c) => {
  const users = await c.env.users.getAll()
  return c.json(users)
})

post('/api/users', async (c) => {
  const data = await c.req.json()
  const user = await c.env.users.create(data)
  return c.json(user, 201)
})
```

## Route patterns

```js
get('/users', handler)
get('/api/status', handler)
get('/users/:id', handler)              // matches /users/123
get('/posts/:slug/comments', handler)   // matches /posts/hello/comments
get('/files/*', handler)                // matches any path under /files
use('/admin/*', middleware)             // matches /admin/users, /admin/settings
```

Patterns are deliberately simple, mapping to Cloudflare routing capabilities. No regex routes, no parameter validation DSL.

## Middleware

Linear execution with explicit `next()`:

```js
// scoped middleware
use('/admin/*', async (c, next) => {
  const auth = c.req.header('authorization')
  if (!auth) return c.json({ error: 'Unauthorized' }, 401)
  await next()
})

// global middleware
use(async (c, next) => {
  console.log(c.req.method, c.req.url)
  await next()
})
```

## The context object (c)

Every handler receives a single context:

```js
get('/example', async (c) => {
  const id = c.req.param('id')          // route parameter
  const page = c.req.query('page')      // single query parameter
  const params = c.req.query()          // all query parameters as object
  const auth = c.req.header('authorization')
  const data = await c.req.json()       // parse JSON body
  const text = await c.req.text()       // parse plain text body
  // also available: c.req.method, c.req.url
})
```

Responses:

```js
return c.json(data)            // 200 JSON
return c.json(data, 201)       // JSON with status code
return c.text('OK')            // plain text
```

Edge bindings (databases, KV, queues) hang off `c.env`:

```js
post('/api/contact', async (c) => {
  const payload = await c.req.json()

  if (!payload.email) {
    return c.json({ error: 'Email required' }, 400)
  }

  await c.env.database.insert(payload)
  return c.json({ success: true }, 201)
})
```

## Cloudflare header mocking

During local development with Nuekit, Cloudflare headers are mocked so edge-dependent code works without deploying:

```js
post('/api/contact', async (c) => {
  const country = c.req.header('cf-ipcountry')
  const ip = c.req.header('cf-connecting-ip')
  const data = await c.req.json()
  return c.json({ ...data, country, ip })
})
```

## Constraints (the compiler and conventions enforce these)

1. No HTML responses. `c.json()` and `c.text()` only. HTML generation belongs to the frontend layer.
2. No file serving. Static assets are handled by the build system, never the server layer.
3. No middleware chaining libraries. Linear `use()` with explicit `next()` only.
4. No Express/Hono/Koa imports of any kind.

## Server proxy

If a project is not ready for edge-first, Nue can proxy API calls to an existing backend instead. Configure the proxy in site.yaml rather than writing fetch-forwarding routes by hand.
