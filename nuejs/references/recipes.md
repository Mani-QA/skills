# Recipes - complete worked examples

Three end-to-end builds. Copy the shapes, not just the snippets.

## Recipe 1: Single-page application from scratch

```bash
bun install --global nuekit
nue create spa
cd spa
nue        # http://localhost:4000
```

Generated structure:

```
├── index.html     # SPA entry point
├── site.yaml      # global config
├── ui/            # reusable UI components
├── css/           # design system
└── server/        # backend routes (index.js)
```

### index.html (application shell + routing)

```html
<!doctype dhtml>
<html lang="en">
<head>
  <title>Task App</title>
</head>

<script>
  import { state } from 'state'

  state.setup({
    route: '/app/:view/:id',
    query: ['filter', 'search'],
    local: ['theme'],
    autolink: true
  })
</script>

<body>
  <main>
    <my-dashboard/>
  </main>

  <script>
    mounted() {
      state.init()
    }
  </script>
</body>
</html>
```

### ui/dashboard.html (interactive component)

```html
<my-dashboard>
  <header class="app-header">
    <h1>Welcome, ${ user.name }</h1>
    <button :click="logout">Logout</button>
  </header>

  <ul class="task-list">
    <li :for="task in tasks" class="{ completed: task.isDone }">
      ${ task.title }
      <button :click="toggleTask(task)">Toggle</button>
    </li>
  </ul>

  <script>
    async mounted() {
      const res = await fetch('/api/tasks')
      this.tasks = await res.json()
      this.user = { name: 'Mani' }
      this.update()
    }

    toggleTask(task) {
      task.isDone = !task.isDone
      this.update()
    }

    logout() {
      state.view = 'login'
    }
  </script>
</my-dashboard>
```

### server/index.js (API)

```js
const tasks = [
  { id: 1, title: 'Write tests', isDone: false },
  { id: 2, title: 'Review PR', isDone: true }
]

get('/api/tasks', async (c) => {
  return c.json(tasks)
})

post('/api/tasks', async (c) => {
  const data = await c.req.json()
  if (!data.title) return c.json({ error: 'Title required' }, 400)
  const task = { id: tasks.length + 1, title: data.title, isDone: false }
  tasks.push(task)
  return c.json(task, 201)
})
```

### css/task-list.css (design system fragment)

```css
@layer components {
  .task-list {
    display: grid;
    gap: var(--gap, 0.5rem);
    list-style: none;
    padding: 0;

    > li {
      display: flex;
      justify-content: space-between;
      padding: 0.75em 1em;
      border-radius: var(--radius);
      background: var(--color-surface);

      &.completed {
        opacity: 0.55;
        text-decoration: line-through;
      }
    }
  }
}
```

## Recipe 2: Markdown blog with layout

```bash
nue create blog
```

### site.yaml

```yaml
meta:
  title: Engineering Notes
  description: A blog about QA automation

inline_css: true
view_transitions: true
```

### layout.html (root)

```html
<header>
  <nav>
    <a href="/">Home</a>
    <a href="/posts/">Posts</a>
  </nav>
</header>

<main>
  <slot for="content"/>
</main>

<footer>
  <p>{ meta.title } © 2026</p>
</footer>
```

### posts/playwright-tips.md

```md
---
title: Five Playwright patterns that cut flake
date: 2026-06-11
---

--- #hero
# Five Playwright patterns that cut flake
---

Intro paragraph here.

[.info]
  These patterns assume the native TypeScript runner.

``` typescript
+ await expect(row).toBeVisible()
- await page.waitForTimeout(2000)
```
```

The Markdown renders into `<slot for="content"/>`. The fenced block uses Nueglow: `+` renders green (added), `-` renders red (removed).

## Recipe 3: Shopping cart component (full Hyper feature tour)

```html
<shopping-cart>
  <header>
    <h2>Your Cart</h2>
    <span :if="items.length === 0">Cart is empty</span>
  </header>

  <ul class="cart-list">
    <li :for="{ id, name, price, quantity } in items">
      ${ name } - $${ price }
      <div class="controls" --gap="0.5rem">
        <button :click="increment(id)">+</button>
        <span>${ quantity }</span>
        <button :click="decrement(id)">-</button>
      </div>
    </li>
  </ul>

  <footer>Total: $${ total() }</footer>

  <script>
    constructor(data) {
      this.items = data.items || []
    }

    increment(id) {
      const item = this.items.find(i => i.id === id)
      if (item) item.quantity++
    }

    decrement(id) {
      const item = this.items.find(i => i.id === id)
      if (item && item.quantity > 0) item.quantity--
    }

    total() {
      return this.items.reduce((sum, i) => sum + i.price * i.quantity, 0)
    }
  </script>
</shopping-cart>
```

Usage with data binding:

```html
<shopping-cart :items="cartItems"/>
```

Features demonstrated: destructured `:for`, `:if` empty state, colon event binding with arguments, CSS variable attribute (`--gap`), `constructor(data)` defaulting, computed value via plain method call in interpolation. Note what is absent: no class soup, no styles, no imports, no export.
