# CSS design system - complete reference

A design system in Nue is not a component library. It is a visual language expressed through CSS variables, nesting, and cascade layers. Components carry semantic tags and at most a few descriptive class names; everything visual cascades from the design system. Swapping the entire visual identity means changing CSS variables, not editing components.

## What is forbidden

- Tailwind and all utility classes (`flex`, `pt-4`, `bg-blue-500`)
- CSS-in-JS in any form
- `<style>` blocks inside components (ignored by the compiler)
- `style` attributes (ignored by the compiler)
- More than 3 class names on one element (warning now, error in production strict mode)

## What to use instead

Modern native CSS:

- CSS variables: `var(--color-primary)`
- Native nesting
- Cascade layers: `@layer settings, components, containers, utilities`
- Container queries
- `:has()`, `color-mix()`, logical properties

Selectors target semantic tags first (`article`, `nav`, `dialog`, `time`) and minimal descriptive classes second (`.gallery`, `.card`, `.task-list`).

## Directory organization

The design system lives in `@shared/design/` (auto-loaded client-side) and splits into logical files:

```
@shared/design/
├── settings.css      /* variables, resets        @layer settings */
├── colors.css        /* palette variables */
├── elements.css      /* base element styling */
├── components/       /* card.css, button.css ... @layer components */
├── containers.css    /* layout primitives        @layer containers */
└── utilities.css     /* the few sanctioned utils @layer utilities */
```

Larger multi-app projects separate global foundations from reusable libraries and app-specific styles:

```
website/
├── @global/          # site-wide foundations (declared in site.yaml globals)
├── @components/      # reusable CSS library (declared in site.yaml libs)
├── blog/             # app-specific: settings.css, components/, containers.css
└── app/              # another app with its own overrides
```

```yaml
# site.yaml
globals: ["@global"]       # auto-included on every page
libs: ["@components"]      # opted into per page/app via include
inline_css: true           # inline styles into the page for instant FCP
```

Per app or page, pull in library styles selectively:

```yaml
# app.yaml or front matter
include: [card, form-inputs]   # partial name match against libs content
```

Stylesheets are auto-linked based on directory placement: a CSS file next to a page applies to that page; CSS in an app folder applies to that app; CSS in globals applies everywhere. System-level assets load first.

## Variables as the single source of truth

```css
@layer settings {
  :root {
    --color-primary: oklch(0.62 0.19 260);
    --color-surface: #fff;
    --radius: 0.5rem;
    --gap: 1rem;
    --font-body: system-ui, sans-serif;
  }
}
```

```css
@layer components {
  .card {
    background: var(--color-surface);
    border-radius: var(--radius);
    display: grid;
    gap: var(--gap);

    > header {
      font-weight: 600;
    }
  }
}
```

## Parameterizing components from markup

Components pass styling parameters through CSS variable attributes, never inline styles:

```html
<div class="gallery" --gap="2rem">...</div>
```

```css
.gallery {
  display: grid;
  gap: var(--gap, 1rem);
}
```

## Performance defaults

Set `inline_css: true` to inline styles into the document head, removing the external CSS request and producing instant First Contentful Paint. Keep individual CSS files small and focused; the build system concatenates and minifies in production.
