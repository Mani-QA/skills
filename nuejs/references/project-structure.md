# Project structure, routing, configuration and build system

## File-based routing

Directory structure maps directly to URLs. Any HTML or Markdown file becomes a page. CSS files become stylesheets. Everything else passes through unchanged.

```
index.html          → /
about.html          → /about
blog/index.html     → /blog/
blog/first-post.md  → /blog/first-post/
```

The simplest valid project is a single `index.html`. No configuration, no scaffolding. A content site adds shared layouts:

```
├── site.yaml
├── layout.html        # global header and footer
├── index.css
├── index.md
└── posts/
    ├── header.html    # page hero layout
    ├── first.md
    └── second.md
```

Nue 2.0 serves files directly from the source directory during development. There is no temporary `.dist/dev` folder and no build step before you start working.

## Doctype-based file processing

All HTML files use the `.html` extension. The doctype declaration determines processing:

```html
<!doctype html>     <!-- Server-rendered static page -->
<!doctype dhtml>    <!-- Dynamic, client-rendered page -->
<!html lib>         <!-- Server-side component library -->
<!dhtml lib>        <!-- Client-side component library -->
<!html+dhtml>       <!-- Isomorphic library: identical code runs server and client -->
```

When the doctype is omitted, Nue auto-detects: files with event handlers or imports become dynamic; pure content structure renders server-side. This replaces the old `.dhtml` extension system from Nue 1.0.

## The @shared system directory

Fixed-name directories with special behavior:

```
@shared/
├── design/    # CSS design system (auto-loaded client-side)
├── ui/        # UI components, server and client (auto-loaded client-side)
├── data/      # YAML data for HTML templates (server-side processing)
└── server/    # Backend code (not shipped as frontend assets)
```

Conventional (not hardcoded) additions:

```
@shared/
├── lib/       # Third-party libraries, selectively imported
└── app/       # Business logic / data models (imported)
```

Wire `lib/` and `app/` through import maps in site.yaml:

```yaml
# site.yaml
import_map:
  app: /@shared/app/index.js
  lib: /@shared/lib/
```

```js
import { login } from 'app'      // resolves to @shared/app/index.js
import * as d3 from 'lib/d3'     // resolves to @shared/lib/d3.js
```

The `ui/` and `lib/` directories may contain `.html`, `.js`, `.ts`, and component-specific `.css` files. System-level assets load before root-level assets.

Page-specific assets can live next to the page:

```
blog/css-is-awesome/
├── effects.css     # Only applies to this page
├── awesome.html    # Page-specific components
└── products.yaml   # Page-specific data
```

## Hierarchical configuration

Three levels, each extending and overriding the previous:

1. `site.yaml` at the project root: global metadata, globals, libs, import maps, system settings.
2. `<appdir>/app.yaml`: application-level overrides for that subdirectory (for example enabling SVG processing or changing the title for one section).
3. Markdown front matter: page-level overrides for a single file.

Common site.yaml content:

```yaml
meta:
  title: My Nue Application
  description: Site-wide description

globals: ["@global"]       # directories auto-included on all pages
libs: ["@components"]      # CSS library dirs used by include statements
inline_css: true           # inline CSS into the page for fast FCP
view_transitions: true
```

Data defined in YAML flows into layouts via interpolation. If site.yaml contains `slogan: Less is more`, a layout can render it with `{ slogan }`.

## Layout system (content sites)

Layouts are pure HTML wrappers populated through slots. Create `layout.html`:

```html
<header>
  <nav>
    <a href="/">Home</a>
    <a href="/about">About</a>
  </nav>
</header>

<main>
  <slot for="content"/>
</main>

<footer>
  <p>{ slogan }</p>
</footer>
```

Markdown content is parsed by Nuemark and injected into `<slot for="content"/>`. Predefined layout module slots:

- `banner` - temporary announcements above the global header
- `header` / `footer` - site-wide branding and navigation
- `main` - primary content area
- `aside` - documentation sidebars, contextual info
- `pagehead` / `pagefoot` - content-specific hero areas or calls to action

Prefer modern native HTML in layouts: `<dialog>` with popover attributes, native `<form>` validation. No JavaScript polyfills.

## SVG development

Enable in app.yaml:

```yaml
svg:
  process: true
```

SVG files are then treated as Nue templates with full HMR: embed design system fonts and styles, inject data conditionally, and mix HTML inside SVG. `<html>` tags inside SVG become `<foreignObject>` automatically.

## Hot reload and build behavior

Universal HMR applies to content, CSS, layouts, data, components, server routes, and configuration:

- Markdown: DOM diffing preserves form inputs and scroll position
- HTML/components: smart reload preserves component state
- CSS: styles injected or removed instantly
- YAML/JS: full page reload (these affect fundamental logic)

`nue build` compiles for production. Builds typically complete in well under a second for medium sites. Useful flags: `-p / --production` (production version and stats), `-e / --environment <file>` (override site.yaml options), `-n / --dry-run` (show what would build), `-P / --port` (serve port). You can target file patterns: `nue build .ts .css` builds only TypeScript and CSS files.

Generated extras when enabled in config: `sitemap.xml` from pages (skipping `draft` or `private` flagged pages) and RSS feeds from any content collection with auto-discovery link tags.

## Runtime requirements

Bun 1.2+ only. Node.js is not supported. Bun is chosen because it implements the browser APIs Nue relies on (`fetch()`, `Request`, `Response`, `URL`, `Headers`, `FormData`) and bundling/serving is native Zig code, removing the need for Vite or ESBuild. Nue 2.0 is beta software tested on macOS; Linux and Windows compatibility is not guaranteed. On Windows, run inside WSL.
