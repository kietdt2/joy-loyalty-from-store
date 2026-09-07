# Brand extraction

Turn the analysis into a **token sheet**: role-named, measured, and locked for
the rest of the build. Everything downstream reads these tokens and nothing
downstream re-samples the store.

## Which source wins

1. **`config/settings_data.json`** from the supplied theme folder. Authoritative.
   The merchant's own chosen values.
2. **`config/settings_schema.json`** for what each id means, since ids differ
   per theme family.
3. **Computed styles on the live page.** Only when there is no theme folder.
4. **Screenshot colour sampling.** Last resort, and flag it as approximate.

Never mix: if the theme folder is present, do not overwrite a `settings_data`
value with something sampled off a screenshot.

## From a theme folder

`settings_data.json` holds `current.color_schemes` on OS 2.0 themes. Each scheme
carries `background`, `text`, `button`, `button_label`, `shadow` and often
`secondary_background`. Take the scheme the merchant uses for their main
content, not scheme 1 by default.

Also read from `current`: `type_header_font`, `type_body_font`, `heading_scale`,
`body_scale`, `buttons_radius`, `buttons_border_thickness`,
`card_corner_radius`, `page_width`, `spacing_sections`, `spacing_grid_vertical`.

Font handles look like `assistant_n4`. Resolve the family name from the handle;
never `@font-face` a licensed family and never pull one from a CDN. If the
family is a Shopify-hosted font the theme already loads, inherit it. Otherwise
substitute the nearest available family and flag the substitution in the report.

## From a live page

In the console, read computed styles off real elements rather than off `:root`,
because theme variables are often scoped to `.color-scheme-*`:

```js
const g = (sel, ...props) => {
  const el = document.querySelector(sel); if (!el) return null;
  const cs = getComputedStyle(el);
  return Object.fromEntries(props.map(p => [p, cs.getPropertyValue(p).trim()]));
};
({
  body:    g('body', 'background-color', 'color', 'font-family', 'font-size', 'line-height'),
  heading: g('h1, h2, .h1, .h2', 'font-family', 'font-size', 'font-weight', 'letter-spacing', 'text-transform'),
  button:  g('button, .button, [type=submit]', 'background-color', 'color', 'border-radius', 'border-width', 'padding', 'font-size', 'text-transform'),
  card:    g('.card, .card-wrapper, .product-card', 'border-radius', 'border-width', 'box-shadow', 'background-color'),
  width:   g('.page-width, main .container', 'max-width', 'padding-left')
})
```

Sample the same properties on the home, collection and product pages. Where they
disagree, the value that appears on two of three pages is the theme's real
default.

## The token sheet

Role names, not colour names. `--joy-surface`, never `--joy-cream`. The whole
point is that the page can be re-themed without renaming anything.

```
TOKEN SHEET - <brand>            source: <settings_data.json | live page | mixed>

colour
  --joy-bg                 #......   page background
  --joy-surface            #......   card / panel
  --joy-surface-alt        #......   alternating band
  --joy-text               #......   body
  --joy-text-muted         #......   secondary
  --joy-primary            #......   primary action
  --joy-on-primary         #......   label on primary
  --joy-accent             #......   highlight / tier / badge
  --joy-border             #......
type
  --joy-font-heading       <family>, <fallback stack>     substituted: yes|no
  --joy-font-body          <family>, <fallback stack>
  --joy-h1 / h2 / h3       <px / clamp>, weight, tracking, casing
  --joy-body / small       <px>, line-height
shape
  --joy-radius             <px>      --joy-radius-sm  <px>
  --joy-border-width       <px>
  --joy-shadow             <value or none>
layout
  --joy-page-width         <px>
  --joy-gutter             <px>
  --joy-section-pad        <px desktop / px mobile>
  --joy-gap                <px>
dials  (from the analysis, 1-10)
  variance <n>   motion <n>   density <n>
```

Every token use downstream carries a literal fallback:
`var(--joy-primary, #1a1a1a)`. A missing variable must degrade to something
readable, never to `initial`.

## Contrast

Check every text-on-surface and label-on-primary pair before locking. Body text
needs 4.5:1, large headings and UI 3:1. When the merchant's own combination
fails, keep their hue and adjust lightness to pass, then note the adjustment in
the report. Do not silently ship their inaccessible pair, and do not silently
replace their colour with an unrelated one.

## Handing the sheet on

The sheet ships once as a snippet rendered from the topmost Joy section, so no
band can drift. `joy-loyalty-build` owns that mechanism and its template
(`templates/joy-figma-tokens.liquid`). Fill it there; do not invent a second
token snippet here.
