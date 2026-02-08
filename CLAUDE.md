# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is **Horizon**, a Shopify theme (v3.3.0). It is a complete e-commerce storefront built with Liquid, JavaScript (ES2020), and CSS. There is no build system, bundler, or package manager — files are deployed directly to Shopify's infrastructure.

## Development Commands

There are no build, lint, or test commands. The theme is developed using the [Shopify CLI](https://shopify.dev/docs/themes/tools/cli):

```bash
shopify theme dev          # Start local development server with hot reload
shopify theme push         # Deploy theme to Shopify store
shopify theme pull         # Pull latest theme from store
shopify theme check        # Run Theme Check linter (Liquid/JSON linting)
```

JavaScript type checking uses JSDoc annotations with TypeScript's compiler (configured in `assets/jsconfig.json`):

```bash
cd assets && npx tsc --noEmit   # Type-check JS files using jsconfig.json
```

## Architecture

### Shopify Theme Structure

- **`layout/`** — Master HTML templates. `theme.liquid` is the main entry point that wraps all pages.
- **`templates/`** — JSON page templates (product, collection, index, cart, etc.) that compose sections.
- **`sections/`** — Major customizable page regions with `{% schema %}` blocks defining their settings.
- **`blocks/`** — Reusable sub-components within sections. All prefixed with `_` (e.g., `_card.liquid`).
- **`snippets/`** — Liquid partials included via `{% render %}`. Many use `{%- doc -%}` tags to document parameters.
- **`config/`** — `settings_schema.json` defines admin-facing customization options; `settings_data.json` stores current values.
- **`locales/`** — 50+ language files. `en.default.json` is the source of truth. Translation keys use `t:` prefix in Liquid.
- **`assets/`** — JavaScript modules, CSS, images, and `jsconfig.json`.

### JavaScript Component System

All JS lives in `assets/` as ES6 modules using the `@theme/*` path alias (maps to `assets/*` via jsconfig.json).

**Core files:**
- **`component.js`** — Defines `Component` (extends `DeclarativeShadowElement`), the base class for all web components. Provides automatic `ref` tracking (via `[ref]` attributes and MutationObserver) and declarative event delegation (via `on:{event}` attributes).
- **`events.js`** — Custom event classes (`VariantUpdateEvent`, `CartUpdateEvent`, etc.) and the `ThemeEvents` namespace for inter-component communication via `document.dispatchEvent()`.
- **`utilities.js`** — Shared helpers: `requestIdleCallback`, `yieldToMainThread`, view transition support, header height calculations.
- **`section-renderer.js`** — `SectionRenderer` class for re-rendering sections via Shopify's Section Rendering API with DOM morphing.
- **`global.d.ts`** — TypeScript declarations for global `Shopify` and `Theme` objects.

**Component conventions:**
- Components are custom elements named `*-component` (e.g., `<header-component>`, `<cart-drawer-component>`).
- They use Declarative Shadow DOM (`<template shadowrootmode="open">`).
- Event binding uses `on:{event}` attributes: `on:click="closest-component/methodName"` routes events to component methods.
- Child element references use `ref="name"` (single) or `ref="name[]"` (array).
- Type safety via JSDoc with strict TypeScript checking — no `.ts` files, all types are in JSDoc comments.

### CSS Architecture

- `base.css` — Global styles and CSS custom properties.
- Theming uses CSS custom properties set by `snippets/theme-styles-variables.liquid` and `snippets/color-schemes.liquid`.
- Responsive spacing uses `snippets/spacing-style.liquid` which outputs `clamp()`/`max()` CSS values from settings.
- Six built-in color schemes (`color-scheme-1` through `color-scheme-6`) configurable in theme admin.

### Key Patterns

- **Section Rendering API**: Components re-render by fetching section HTML from Shopify and morphing it into the DOM (see `section-renderer.js` and `morph.js`).
- **Translation keys**: Always use `t:` prefixed keys (e.g., `{{ 'products.product.add_to_cart' | t }}`) — never hardcode user-facing strings.
- **Settings visibility**: Schema settings support `visible_if` conditions for conditional display in the admin editor.
- **Inline critical JS**: `layout/theme.liquid` contains inline JavaScript for header height calculations to prevent layout shift — changes must stay in sync with `utilities.js`.
