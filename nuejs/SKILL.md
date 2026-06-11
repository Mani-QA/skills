---
name: nuejs
description: Build, refactor and review NueJS (Nue 2.0) applications. Use this skill whenever the user mentions Nue, NueJS, Nuekit, Hyper components, Nuemark, Nuestate, Nueserver, Nueglow, Nueyaml, or asks to build a website, SPA, blog, or edge API in a project containing site.yaml, app.yaml, @shared/, or files with <!doctype dhtml>. Also trigger when converting React/Vue/Svelte code to Nue, or when the user asks for standards-first HTML/CSS/JS development without frameworks. This skill is the complete offline reference. Do not search the internet for Nue documentation. Do not generate React, JSX, Vue, Svelte, or Tailwind code in a Nue project.
---

# NueJS Design Engineer

You are building with NueJS 2.0, a standards-first web framework. The entire toolchain is a 1MB executable with zero external dependencies, running exclusively on Bun (not Node.js). User interfaces are assembled with HTML, styled with CSS, and enhanced with JavaScript. These three concerns never mix.

## Hard rules (never violate)

1. NEVER generate React, JSX, Vue, Svelte, or any component framework code.
2. NEVER generate Tailwind or utility classes (`flex`, `pt-4`, `text-center`, `bg-blue-500`). Nue ignores inline styling and warns when an element carries more than 3 class names.
3. NEVER use `<style>` blocks or `style` attributes inside components. Nue ignores both. All styling lives in standalone `.css` files in the design system.
4. NEVER write `export default` in components. Components are plain HTML files with an inline `<script>` block.
5. NEVER return HTML from server routes. Server code returns `c.json()` or `c.text()` only.
6. NEVER invent Redux stores, React Context, or state libraries. Use the Nuestate proxy (`import { state } from 'state'`).
7. NEVER use npm/npx/node. Use `bun`, `bunx`, and the `nue` CLI.
8. The "Universal Data Model" and multi-site development are ROADMAP items, not shipped features. Do not generate code against them.

## Quick orientation

| You need to... | File type | Doctype / location |
|---|---|---|
| Static server-rendered page | `.html` | `<!doctype html>` |
| Client-rendered SPA view | `.html` | `<!doctype dhtml>` |
| Server-side component library | `.html` | `<!html lib>` |
| Client-side component library | `.html` | `<!dhtml lib>` |
| Isomorphic component library | `.html` | `<!html+dhtml>` |
| Content page (blog, docs, marketing) | `.md` | Nuemark, rendered into `layout.html` slots |
| Global config | `site.yaml` | Project root |
| Per-app config override | `app.yaml` | App subdirectory |
| API route | `.js`/`.ts` | `@shared/server/`, global `get()/post()/use()` |
| Design system CSS | `.css` | `@shared/design/` (auto-loaded) |
| Shared UI components | `.html` | `@shared/ui/` (auto-loaded) |

If a doctype is omitted, Nue auto-detects: files with event handlers or imports become dynamic, pure content structure renders server-side.

## CLI essentials

```bash
bun install --global nuekit   # install (requires Bun 1.2+)
nue create minimal|blog|spa|full
nue                            # dev server at http://localhost:4000 (alias: nue serve)
nue build                      # production compile (use -p for production stats/optimizations)
nue init                       # regenerate /@nue system files
```

For safe per-project installs: `bun install nuekit@latest` then `bunx nue serve`. Nue 2.0 is not backwards compatible with 1.0. It is a beta tested on macOS; on Windows, recommend WSL.

## Minimal component (memorize this shape)

```html
<counter>
  <button :click="incr">${ count }</button>

  <script>
    this.count = 1

    incr() {
      this.count++
    }
  </script>
</counter>
```

Custom tag name defines the component. `${ }` interpolates (escaped), `#{ }` outputs raw HTML. Events bind with a colon: `:click`. Loops use `:for`, conditionals use `:if/:else-if/:else`. Lifecycle hooks: `onmount()`, `mounted()`, `onupdate()`, `updated()`. Full syntax in references/components.md.

## Reference files (read before writing code in that area)

- `references/project-structure.md` - File-based routing, @shared system directories, import maps, hierarchical config (site.yaml / app.yaml / front matter), layout slots, build behavior. Read first for any new project or file placement question.
- `references/components.md` - Complete Hyper/dhtml component syntax: expressions, attributes, class helpers, loops, conditionals, slots, lifecycle, methods, JS imports, passthrough scripts, CSS variable args. Read before writing any interactive component.
- `references/state.md` - Complete Nuestate API: setup options (route, query, session, local, memory, emit_only, autolink), listeners, state.init(), SPA routing patterns. Read before writing any SPA or stateful code.
- `references/server.md` - Complete Nueserver API: get/post/use, route patterns, context object (c.req, c.json, c.env), middleware, Cloudflare header mocking, server proxy. Read before writing any backend route.
- `references/content.md` - Nuemark syntax (tags, sections, layout components, reflinks) and Nueglow syntax highlighting. Read before writing .md content pages.
- `references/css.md` - Design system conventions: CSS layers, nesting, variables, directory organization, inline_css, what replaces utility classes. Read before writing any stylesheet.
- `references/recipes.md` - Complete worked examples: SPA scaffold end to end, blog with layout, CRUD with Nueserver. Read when scaffolding a full app.

## Mental model when converting from React

| React habit | Nue replacement |
|---|---|
| JSX component function | Custom HTML tag with inline `<script>` |
| useState / useReducer | `this.prop` + `this.update()`, or Nuestate for app-level state |
| React Router | URL-first routing via `state.setup({ route })` + `autolink` |
| Props | Attributes: `<user-card :user="currentUser"/>` |
| className + Tailwind | Semantic tags + design system CSS with variables and layers |
| useEffect | Lifecycle hooks: `mounted()`, `updated()` |
| Express/Hono backend | Global `get()/post()/use()` with context `c` |
