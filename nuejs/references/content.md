# Nuemark and Nueglow - content authoring, complete reference

## Nuemark

Nuemark is Nue's extended Markdown. Content pages (`.md`) compile server-side into the layout's `<slot for="content"/>`. The goal: content creators craft rich pages without touching HTML or JSX. Syntax draws from WordPress shortcodes, TOML, and standard Markdown.

### Front matter

Page-level configuration sits at the top of the file and overrides app.yaml and site.yaml:

```md
---
title: My first post
date: 2026-06-11
draft: false
---
```

### Tags (components in content)

Square-bracket tags invoke built-in or custom components:

```md
// named option
[image src="hello.png"]

// unnamed option
[image hello.png]

// YAML options block
[image]
  caption: Hello, World!
  src: hello.png

// content argument: accepts *formatting* and nested tags
[.info]
  This is a pure content argument

// classes and id
[image#hero.epic.bordered hero.webp]
```

Layout wrapper tags generate semantic grid/flex/stack containers:

```md
[.grid]
  ...columns of content...

[.stack]
  ...stacked content...

[layout.box]
  ...boxed content...
```

Built-in interactive tags include `[image]`, `[! image.png]`, `[tabs]`, `[video]`, `[button]`, `[table]`, `[accordion]`. Custom server-side or isomorphic components registered in the project can be invoked the same way, so authors gain features without seeing HTML.

### Sections

Three hyphens (`---`) split a page into sections (the same token that ends front matter). Class names and ids attach with `#id.foo.bar` notation after the separator:

```md
--- #hero.dark
# Big headline
---
Regular content
```

Section definitions can also be set globally in site.yaml or per app in app.yaml.

### Reflinks

Define site-wide reference links once in site.yaml; use them in any page:

```yaml
# site.yaml
links:
  issues: //github.com/nuejs/nue/issues "Nue Github Issues"
  divide: //css-tricks.com/the-great-divide/ "JS devs vs UX devs"
```

### Content collections

Folders of Markdown files become collections (post lists, doc trees) usable in layouts and RSS feed generation. Enable RSS and sitemap output in site.yaml; pages flagged `draft` or `private` are excluded.

## Nueglow - syntax highlighting

Nueglow parses code aesthetically instead of loading per-language grammar JSON. The engine is under 3KB.

Fenced code blocks support the `language-name#id.class` notation:

```md
``` javascript#example.numbered
function hello() { ... }
```
```

Line-level marks inside fenced blocks:

- `>` prefix highlights a line
- `+` prefix marks a line as added (renders green)
- `-` prefix marks a line as removed (renders red)

Output is semantic HTML styled entirely by the design system CSS:

- `<b>` keywords
- `<em>` values/strings
- `<i>` punctuation
- `<sup>` comments

Style code blocks with the same CSS variables as the rest of the design system; no highlight.js, no Prism, no theme files.
