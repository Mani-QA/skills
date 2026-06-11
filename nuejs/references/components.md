# Component syntax (Hyper / dhtml) - complete reference

Components are standard `.html` files. A custom tag name wraps the markup and an inline `<script>` holds initialization and methods. Components render identically on server and client. Place reusable components in `@shared/ui/` or in a library file with a `lib` doctype.

## Enforced separation of concerns

The compiler enforces these. Generated code violating them is silently ignored or warned about:

1. `<style>` blocks inside components are ignored.
2. `style` attributes are ignored.
3. Inline styling through class names is not permitted; expressions like `size-[max(100%,2.75rem)]` are ignored.
4. Default limit of 3 class names per element. Exceeding it triggers a warning (strict mode in production will throw).

All visual design lives in the CSS design system. See references/css.md.

## Expressions

```html
<span>${ text }</span>

<p>${ text.toUpperCase() }</p>

<!-- raw, non-escaped HTML output -->
<p>#{ value }</p>
```

## Attributes

```html
<time datetime="${ date.toISOString() }">

<!-- boolean attributes: falsy properties are omitted from output -->
<button disabled="${ is_disabled }">

<!-- interpolation inside attribute values -->
<div class="gallery ${ class }">

<!-- class helper: object constructor syntax, no $ prefix -->
<label class="{ is-active: isActive, has-error: hasError }">

<!-- combined -->
<div class="gallery ${ type } { is-active: isActive() }">
```

## Loops (:for)

```html
<li :for="el in items">${ el.text }</li>

<!-- with index -->
<li :for="(el, i) in items">
  Text: ${ el.text }
  Index: ${ i }
</li>

<!-- destructuring -->
<li :for="{ lang, text } in items">
  ${ lang } = ${ text }
</li>

<!-- destructuring with index -->
<li :for="{ lang, text }, i in items"> ... </li>

<!-- nested loops with parent access -->
<li :for="item in items">
  <p :for="child in item.children">
    ${ item.text } ${ child.text }
  </p>
</li>

<!-- object entries -->
<li :for="[lang, text] in Object.entries(items)">
  ${ lang } = ${ text }
</li>

<!-- loop multiple elements without a wrapper -->
<dl>
  <template :for="el in meta">
    <dt>${ el.title }</dt>
    <dd>${ el.data }</dd>
  </template>
</dl>
```

Both `in` and `of` work: `:for="user of users"` is valid.

## Conditionals

```html
<p :if="foo > 100">${ foo }</p>
<p :else-if="bar == 10">${ foo }</p>
<p :else>Baz</p>

<!-- conditional + loop on same element: condition takes precedence -->
<li :if="todos.length < 10" :for="todo in todos"> ... </li>
```

## Component definition, state and methods

```html
<!-- definition: custom tag wraps everything -->
<counter>
  <button :click="count++">${ count }</button>

  <!-- initialization script -->
  <script>
    this.count = 1
  </script>
</counter>

<!-- usage -->
<div>
  <h1>Counter</h1>
  <counter/>
</div>

<!-- passing data via attributes -->
<counter :count="10"/>
```

Methods are defined inside the script with object-method shorthand or `this` assignment. Never `export default`.

```html
<counter>
  <button :click="incr">${ count }</button>

  <script>
    // object-style method (preferred)
    incr() {
      this.count++
    }
  </script>
</counter>
```

Direct event expressions also work: `:click="count++"`. Pass the DOM event explicitly when needed:

```html
<button :click="log('Hey', $event)">Hey</button>
```

To force re-rendering after mutating data inside a method, call `this.update()`. To mount a different component into a placeholder element, use `this.mount(componentName, selectorOrName)`.

Define a component on a specific root element (default is `div`) with `:is`:

```html
<figure :is="image">
  ...
</figure>

<!-- usage on a standard tag -->
<time :is="pretty-date" datetime="${ date.toISOString() }">
  ${ pretty }
  <script>
    this.pretty = prettyDate(this.date)
  </script>
</time>
```

## Events

Bind with a colon prefix: `:click`, `:input`, `:submit`, `:keydown`, and so on. The `:on` prefixed form is also accepted (`:onclick`, `:oninput`). The handler receives `$event` when passed explicitly.

```html
<input value="{ state.search }" :oninput="state.search = $event.target.value">
```

## Slots

```html
<!-- component with a slot -->
<hello>
  <h3>Hello</h3>
  <slot/>
</hello>

<!-- usage: nested content replaces <slot/> -->
<hello>
  <h4>World!</h4>
  <p>Lets go</p>
</hello>

<!-- slots combined with loops -->
<hello :for="item in items">
  <h4>${ item.text }</h4>
</hello>

<!-- :bind exposes item properties directly inside the component -->
<hello :for="item in items" :bind="item">
  <h4>${ text }</h4>
</hello>
```

## Lifecycle hooks

Declared inside the script block:

```html
<counter>
  <button :click="incr">${ count }</button>

  <script>
    incr() { this.count++ }

    onmount() {}    // before mounting
    mounted() {}    // after mounting (can be async)
    onupdate() {}   // before update
    updated() {}    // after update
    unmounted() {}  // after removal
  </script>
</counter>
```

`constructor(data)` runs first and receives the data passed via attributes:

```html
<script>
  constructor(data) {
    this.items = data.items || []
  }
</script>
```

## JavaScript imports

```html
<script>
  import { prettyDate } from './utils.js'
</script>
```

Note: import statements inside components do not execute server-side in the current developer preview. Keep imported logic client-only or move it to `@shared/app/` modules.

## Passthrough JavaScript

Script tags carrying a `type` or `src` attribute are passed to the client verbatim instead of being compiled as component logic:

```html
<footer>
  <script type="module">
    console.info({ hello: 'World' })
  </script>
</footer>
```

## CSS variable arguments

Pass styling parameters to CSS without inline styles. An attribute starting with `--` renders as a scoped CSS variable:

```html
<!-- renders as style="--gap: 3px" -->
<div --gap="3px">...</div>
```

The design system consumes it: `gap: var(--gap, 1rem)`.

## Standalone Hyper usage (outside Nuekit)

Rarely needed, but the library can run alone:

```js
import { compile, render } from 'nue-hyper'

const js = compile('<h1>Hello, ${ name }!</h1>')          // → client JS string
const html = render('<h1>Hello, ${ name }!</h1>', { name: 'World' })  // → SSR HTML
```

Client mounting:

```html
<main></main>
<script type="module">
  import { createApp } from '/dist/hyper.js'
  import { lib } from '/builds/hello.js'
  const app = createApp(lib, { name: 'Hyper' })
  app.mount(document.querySelector('main'))
</script>
```

JIT variant compiles `<script type="text/hyper">` templates directly in the browser via `hyper-jit.js`. Inside Nuekit projects none of this is needed; the framework compiles and mounts automatically.
