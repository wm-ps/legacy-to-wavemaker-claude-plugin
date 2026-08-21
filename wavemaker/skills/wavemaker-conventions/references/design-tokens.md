# Design tokens & component variants

Covers the design-token theme implementation and the component-variant ("appearances") styling
mechanism. (Sections §8–§9 of the original ruleset.) Companion files:
[`pages-and-markup.md`](pages-and-markup.md), [`data-variables.md`](data-variables.md),
[`security.md`](security.md), [`migration-map.md`](migration-map.md).

---

## 8·0. FIRST: pull the palette FROM the legacy app (don't invent colors)

A migration must **match the legacy app's look**, so derive the tokens from the legacy source
rather than picking a fresh palette. Before writing any token file, extract the real values:

- **Find the legacy theme colors.** Grep the legacy static assets for the brand/primary/accent
  colors and fonts (Bootstrap/Colorlib themes keep them in SCSS variables or a theme CSS):
  ```bash
  # hex colors, most-used first
  grep -rhoE '#[0-9a-fA-F]{6}\b' legacy/**/static/{css,scss} | sort | uniq -c | sort -rn | head
  # named SCSS/CSS variables (primary/secondary/accent/brand)
  grep -rniE '(\$|--)(primary|secondary|accent|brand|theme|main)[-a-z]*\s*:' legacy/**/static/{scss,css}
  # font families
  grep -rhoE 'font-family:[^;]+;' legacy/**/static/{css,scss} | sort | uniq -c | sort -rn | head
  ```
  Also check the theme's Google-Fonts `<link>`/`@import` in the legacy `<head>`/`head.jsp`, and any
  button/price/link colors (the accent is usually the "add to cart"/price/sale color).
- **Map them onto the WaveMaker tokens**: legacy brand/header color → `--wm-color-primary`; the
  accent (CTA/price/links) → `--wm-color-secondary`; page bg → `--wm-color-background`; card bg →
  `--wm-color-surface`; muted text → `--wm-color-on-surface-variant`; hairlines → `--wm-color-border`.
  Legacy `font-family` → `--wm-font-family-base`. Convert 6-digit hex to 8-digit `#rrggbbaa`.
- **State the mapping** to the user (`legacy #xxxxxx → --wm-color-primary`, …) so it's reviewable,
  and only fall back to a neutral palette if the legacy source genuinely has no theme colors — and
  say so explicitly.
- **Chrome backgrounds are PER-SECTION — do NOT paint the header/footer/hero with `--wm-color-primary`.**
  A brand accent (e.g. a purple used for buttons + active nav) is NOT the header colour. Inspect the
  legacy page and read each band's ACTUAL background: storefronts commonly have a **white header**, a
  **distinct hero colour** (teal/mint/image), and a **dark/black footer**, with the accent showing only
  on buttons, price, and the active nav link. Capture each as its own token
  (`--wm-color-hero`, `--wm-color-footer`, …) + its own container appearance; give the white/dark bars
  the right text colour (dark-on-white, light-on-dark). Verify by loading the legacy site and reading
  computed `background-color` per band — mapping everything to `primary` produces an all-accent app
  that looks nothing like the original.
- Then write the values into **both** `app.override.css` `:root` and
  `overrides/global/color/color.light.json` (§8a). The same rule applies to `space`/`radius`/`font`
  if the legacy theme has a distinctive scale.

## 8. Theme / design tokens implementation (`src/main/webapp/design-tokens/`)

`index.html` loads `design-tokens/app.override.css` (do **not** edit index.html — the reference is
already there). That **compiled CSS is what styles the app at runtime**; the `overrides/**/*.json`
files are the **source tokens** Studio's theme editor reads. When hand-editing, change **both** the
compiled CSS and the matching JSON so Studio and runtime stay in sync.

### 8·React — Studio REGENERATES app.override.css; custom CSS + web-fonts belong in app.css
Verified live on the React/PRISM runtime (platform.wavemaker.ai) — three hard-won rules:

1. **On import, Studio recompiles `design-tokens/app.override.css` from `overrides/**` and DROPS
   anything you hand-wrote there that isn't token-derived** — your custom classes (`.app-nav-link`,
   `.btn-accent`, `.app-hero`, …) and any `@import url(...)` web-font are erased. The
   `--wm-color-*` vars survive (compiled from `color.light.json`). → **Put custom CSS classes AND the
   Google-Fonts `@import` in `app.css`** (which Studio does NOT regenerate and index.html loads at
   runtime); have those rules read `var(--wm-color-*)`. Do NOT rely on hand-written rules living in
   app.override.css — they will not survive the first import.
2. **The React runtime renders widgets as Material-UI (MUI).** Invented component-appearance names do
   **not** render — `variant="accent"` / `variant="outlined-accent"` produced unstyled buttons. Use
   the built-in variants: **`variant="filled:primary"`** (solid brand colour via the MUI primary
   token) and `variant="outlined"`. A widget's `class=` still lands on its root element, so
   class-based styling in app.css works — but drive colour through the built-in variant + primary token.
3. **Set the base font in `font.config.js` (`baseFont: '<Family>'`), not only in CSS** — the runtime
   feeds `baseFont` into the MUI theme (MUI otherwise defaults to Roboto). Combine with the app.css
   `@import` that loads the actual font files; a broad `#app-root [class*="Mui"] { font-family: … }`
   rule forces it across MUI components.

Folder structure (mirror exactly — same as the provided `design-tokens (3)` sample):
```
design-tokens/
  app.override.css                       # :root token vars + component overrides (loaded at runtime)
  overrides/
    global/{color,font,space,radius,border,elevation,opacity,icon}/<name>.json
    components/{button,form-controls,chips,data-table,modal-dialog,page-top-nav,page-left-nav}/<name>.json
```

### 8a. Changing the palette (edit BOTH files)
Colors live in `app.override.css` `:root { --wm-color-*: ... }` **and**
`overrides/global/color/color.light.json`. Values are 8-digit hex `#rrggbbaa`.
Key tokens: `--wm-color-primary`, `--wm-color-secondary` (accent), `--wm-color-background`,
`--wm-color-on-background`, `--wm-color-surface`, `--wm-color-surface-container`,
`--wm-color-on-surface`, `--wm-color-on-surface-variant`, `--wm-color-border`, `--wm-color-outline`.

`color.light.json` shape (note the `"@": { "value": ... }` wrapper; nested tokens like
`surface.container` nest again):
```json
{ "color": {
  "primary":   { "@": { "value": "#23272bff" } },
  "secondary": { "@": { "value": "#e14b3bff" } },
  "surface":   { "@": { "value": "#ffffffff" }, "container": { "@": { "value": "#f4f5f6ff" } } }
} }
```

### 8b. Use tokens in page/partial CSS (don't hardcode)
```css
.app-product-card .app-price { color: var(--wm-color-secondary); }
.app-card { border: 1px solid var(--wm-color-border); border-radius: var(--wm-radius-md);
            box-shadow: var(--wm-elevation-shadow-4); }
```

### 8c. Other token groups (all `:root` vars in app.override.css)
`--wm-space-*` (rule = 4px base), `--wm-radius-*` (none/xxs/xs/sm/md/lg/xl/circle),
`--wm-font-family-*` + `--wm-h1..h6-*` + `--wm-body-*`, `--wm-elevation-shadow-1..5`,
`--wm-opacity-hover/focus/active`, `--wm-icon-size-*`. Component blocks are class-scoped and read the
globals, so a global color change cascades: `.btn-btn-main-action`/`.btn-btn-accent`/`.btn-btn-outlined`
(buttons), `.top-block` (top-nav, height 62px), `.aside-left-block` (left nav), `.form-block`
(inputs), `.data-table-block`, `.modal-legacy-dialog`, `.app-chips`.

---

## 9. Component variants ("appearances") — the real styling mechanism

The polished/legacy-matching pages style widgets with **named component variants**, not ad-hoc CSS.
Each styled widget carries **both** `variant="<name>"` **and** `class="<component>-<name>"`:

```html
<wm-label  caption="Finding Your Perfect Shoes" class="label-hero-title"  variant="hero-title"></wm-label>
<wm-button caption="SHOP NOW"                    class="btn-shop-cta"      variant="shop-cta"></wm-button>
<wm-container ... class="container-hero"          variant="hero"></wm-container>
<wm-list   ... class="list-product-card"          variant="product-card"></wm-list>
<wm-icon   ... class="icon-feature"               variant="feature"></wm-icon>
```
Plain (unstyled) containers use `class="app-container-default" variant="default"`.

**Where variants are defined:** `design-tokens/overrides/components/<component>/<component>.json`
under `<component>.appearances.<variant>.mapping`. Each mapping entry is a CSS property →
`{ "value": ..., "type": "color|font|space" }`, where `value` is a **global-token reference**
`{group.path.@.value}` (e.g. `{color.on-surface.@.value}`, `{color.on-surface.variant.@.value}`,
`{font.weight.700.value}`, `{h3.font-size.value}`, `{body.medium.font-size.value}`) **or a literal**
(`#7952ebff`, `40px`, `0.5px`, `1.6`). Example (`components/label/label.json`):
```json
{ "label": { "appearances": {
  "hero-title":     { "mapping": {
      "color":       { "value": "{color.on-surface.@.value}", "type": "color" },
      "font-size":   { "value": "40px", "type": "font" },
      "font-weight": { "value": "{font.weight.700.value}", "type": "font" } } },
  "muted-body":     { "mapping": {
      "color":       { "value": "{color.on-surface.variant.@.value}", "type": "color" },
      "line-height": { "value": "1.6", "type": "font" } } }
} } }
```
Component override files added for the storefront: **`label`, `container`, `list`, `icon`** (alongside
`button`, `form-controls`, `chips`, `data-table`, `modal-dialog`, `page-top-nav`, `page-left-nav`).
Register each new component override file so Studio compiles it into `app.override.css`.
