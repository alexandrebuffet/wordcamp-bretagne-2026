# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This repo produces the **Remote CSS** stylesheet for the WordCamp Bretagne 2026 WordPress site (bretagne.wordcamp.org/2026). It is not a plugin or theme — it's a single SCSS file compiled to CSS via `@wordpress/scripts`, which the site's admin (Appearance → Remote CSS) fetches on each commit to production. There is no PHP, no blocks, no runtime JS beyond the webpack-generated stub.

The stylesheet layers on top of the Twenty Twenty-Five block theme and its `theme.json` design tokens, which live on the WordPress site, not in this repo.

## Commands

```bash
npm start        # wp-scripts start — watch mode, dev build
npm run build     # wp-scripts build — production build to build/
npm run lint:css  # wp-scripts lint-style (stylelint, config: .stylelintrc.json)
```

There is no test suite. `build/` is committed (it's what the Remote CSS webhook reads), so always run `npm run build` before a commit that should ship.

## Architecture

Single entry point: `src/index.js` imports `src/style.scss`, which only pulls in partials via `@use` — everything lives under `src/styles/`:

- `styles/abstracts/` — design tokens and helpers, forwarded together via `abstracts/_index.scss`.
  - `_variables.scss` defines Sass maps (`$wp-preset-colors`, `$wp-preset-font-families`, `$wp-preset-font-sizes`, `$breakpoints`, etc.) that mirror the site's `theme.json` presets, and `wp-preset-*($token)` functions that emit `var(--wp--preset--{type}--{token}, {fallback})`.
  - `_mixins.scss` defines `get-clamp()` for fluid sizing and `breakpoint-min/max($name)` mixins keyed off `$breakpoints`.
- `styles/elements/` — bare HTML element styling (button, heading, link, text-input), not tied to a specific block/plugin.
- `styles/blocks/` — per-plugin/block-family partials, each with an `_index.scss` that forwards its siblings: `core` (WP core blocks, e.g. list-as-menu style variations), `jetpack` (Jetpack blocks), `camptix` (CampTix ticketing plugin — registration tables, attendee list, coupon form).
- `styles/utilities.scss` — single-purpose utility classes, prefixed `is-wcbzh-*` / `has-wcbzh-*` (e.g. `.has-wcbzh-brutalist-shadow`).

### Conventions to follow

- **Never hardcode a color, font, spacing, or radius that has a `theme.json` preset.** Use `wp-preset-color()`, `wp-preset-font-family()`, `wp-preset-font-size()`, `wp-preset-spacing()`, etc. from `abstracts/_variables.scss`, or reference the CSS custom property directly (`var(--wp--preset--color--contrast)`) — both are used interchangeably in the codebase depending on whether a Sass-time value is needed (e.g. inside `color-mix()`, string interpolation).
- **Breakpoints** go through `$breakpoints` (`sm`/`md`/`lg`) and the `breakpoint-min`/`breakpoint-max` mixins, not ad hoc media queries — except for plugin-specific historical breakpoints called out explicitly (e.g. `$camptix-tickets-breakpoint-max`).
- **New custom classes** follow the `is-wcbzh-*` (style/behavior toggle) or `has-wcbzh-*` (property override) naming used in `utilities.scss`.
- Selectors targeting third-party plugin markup (CampTix, Jetpack) intentionally use the plugin's own IDs/classes (`#tix`, `.tix-*`, `.jp-related-posts-i2__post`) since that markup isn't ours to change — don't try to "clean up" these selectors into BEM.
- `samples/*.html` are raw markup captures from the live CampTix ticketing pages (order summary, ticket list, purchased-ticket view). Use them as the ground truth for CampTix DOM structure when writing/adjusting `styles/blocks/camptix/*` — the plugin doesn't ship readable docs for its markup.
- Commented-out blocks in `_tickets.scss` are retained legacy rules kept for reference during the CampTix restyle; don't treat them as dead code to delete without checking history/intent first.
