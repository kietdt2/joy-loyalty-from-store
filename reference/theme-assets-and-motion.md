# Theme assets, reusable sections, and motion

Three things the merchant already owns and you should reuse rather than invent:
their photography, their custom sections, and their motion system. Skipping any
of them produces a page that looks bought rather than made.

## 1. Assets: the export is not where the pictures are

A Shopify theme export ships **UI icons and CSS, not brand imagery**. Logos and
photography live in Shopify Files and are referenced as
`shopify://shop_images/...`, which does not resolve offline.

So: read `config/settings_data.json` for *which* assets exist, then fetch the
actual files from the live storefront.

```bash
# what the merchant configured
python3 -c "import json;c=json.load(open('config/settings_data.json'))['current'];
print({k:c[k] for k in ('logo','favicon','brand_image') if k in c})"

# the real files
curl -sL -A "Mozilla/5.0" https://<domain>/ -o /tmp/home.html
grep -oE "//<domain>/cdn/shop/files/[^\"'?]+\.(jpg|jpeg|png|webp|svg)" /tmp/home.html | sort -u

# product photography, with prices and titles
curl -sL "https://<domain>/products.json?limit=30" -o /tmp/prods.json
```

Append `?width=1200` to any Shopify CDN URL for a sized copy.

**Take the logo as PNG with alpha** when one exists (`brand_image` is often a
transparent mark). That single file gives you both a header logo and the
floating decor described below, with no cutout work.

**Prefer photography with people in it.** A loyalty page selling an experience
needs faces and places, not packshots. `products.json` titles usually say which
is which: "rooftop celebration", "sunset lifestyle", "women pose".

Check every image with Read before using it. Storefront filenames lie, and a
badly cropped hero is worse than none.

### Verify the price band here, not from the theme

`products.json` carries real prices. It is common for a storefront read to
suggest "mid" while the catalogue is C$100 to C$3,000. Price posture drives the
whole design direction, so correct the store analysis record when the numbers
disagree, and say you corrected it.

## 2. Floating brand marks

A merchant's logo mark, scaled up and faded back, is the cheapest way to make a
page feel like theirs. It beats stock shapes because it is literally their
identity.

Rules that keep it tasteful:

- **Opacity 4% to 10%.** Higher reads as a mistake. On dark bands go to the top
  of that range, on light bands the bottom.
- **Oversized and bleeding.** 300 to 450px on desktop, pulled 100 to 160px past
  the section edge. A mark fully inside the band looks like a stray sticker.
- **Rotate slightly**, 6 to 18 degrees, alternating direction between bands.
- **Alternate sides** down the page so the eye is not pulled one way.
- **On dark surfaces**, invert the mark rather than sourcing a second file.
- Always `pointerEvents: none` and `zIndex: 0`, and set `overflow: clip` on the
  section so the bleed is cropped by the band, not the viewport.
- **Scale down for mobile**, roughly 60% of the desktop size, pulled further
  out. A mark that reads as texture at 1200px reads as clutter at 390px.

Do not put a mark in every band. Leave one or two clean so the device stays a
device.

## 3. Reuse the merchant's own sections

Before authoring a new band, list the theme's non-standard sections. Anything
not shipped with the base theme was built or bought for this merchant, matches
their design language by definition, and usually has a settings schema you can
drive from the theme editor.

```bash
ls sections/ | grep -viE '^(main-|header|footer|announcement|apps|cart-|predictive|collection|product|blog|article|page|password|email|featured|image-|multi|rich-text|newsletter|video|slideshow|collage|collapsible|contact|custom-liquid|related|quick-|slider|link-list|pickup|search|share|sidebar)'

# then read each schema
awk '/{% schema %}/,0' sections/<name>.liquid | python3 -c "
import sys,json; t=sys.stdin.read().replace('{% schema %}','').replace('{% endschema %}','')
d=json.loads(t); print(d['name']); print([s.get('id') for s in d.get('settings',[])])
for b in d.get('blocks',[]): print(' block', b['type'], [s.get('id') for s in b.get('settings',[])])"
```

Map each loyalty band to an existing section where one fits:

| Loyalty band | Look for a section that does |
|---|---|
| Member status | animated stat / metric blocks |
| Social proof | testimonial carousel with image, rating, quote, verified |
| How it works | icon + heading + text feature grid |
| Tier perks | outcome or benefit cards |
| Hero | image banner with overlay and two buttons |

When a match exists, say so in the cut list and hand it to `joy-loyalty-build`
as **reuse, not rebuild**. Fewer new files, guaranteed visual consistency, and
the merchant already knows how to edit it.

## 4. Motion: copy the theme's numbers, not its taste

Themes define a motion system in CSS variables. Use their **easing and
durations exactly**, because that is what makes a new page feel native. You may
override the *character* of a hover if the design direction calls for it, but
say so out loud.

```bash
grep -nE "^\s*--(duration|ease|animation)-" assets/base.css
grep -n -A 10 "@keyframes slideIn" assets/base.css
grep -n "animations_hover_elements|animations_reveal_on_scroll" config/settings_data.json
```

Dawn-family themes, which most Shopify themes descend from, define:

| Token | Typical value |
|---|---|
| `--ease-out-slow` | `cubic-bezier(0, 0, 0.3, 1)` |
| `--duration-short` / `default` / `medium` / `long` / `extra-long` | 100 / 200 / 300 / 500 / 600ms |
| reveal keyframe | `translateY(2rem)` + `opacity 0.01` to `1` |
| `animations_hover_elements: 3d-lift` | `rotate(1deg)` + layered shadow over 500ms |

Translating to Framer, where the tween syntax is
`tween <ease> <duration> <delay>`:

```
appearEffect.trigger="onInView" appearEffect.threshold="0.15"
appearEffect.enter.y="32" appearEffect.enter.opacity="0"
appearEffect.enter.transition="tween 0,0,0.3,1 0.6s 0s"

hoverEffect.y="-4px" hoverEffect.shadow="0px 18px 32px -16px rgba(...)"
hoverEffect.transition="tween 0,0,0.3,1 0.5s 0s"
```

`2rem` is 32px. Match the theme's reveal distance rather than inventing one.

**Where the theme and the direction disagree.** A playful theme may hover-rotate
cards; a luxury direction wants a straight lift. Keep the theme's easing and
duration, change only the transform, and record the override in the direction
contract. Never silently drop the theme's motion and substitute your own.

**Widgets are not the theme.** Joy's components carry their own tokens
(`--joy-transition-base: 200ms cubic-bezier(0.4, 0, 0.2, 1)`). Anything drawn to
represent the widget uses Joy's timing, not the merchant's. Two systems on one
page is correct here, because that is what ships.

**Reduced motion.** The theme's `@media (prefers-reduced-motion)` block usually
kills transitions outright. Match that: any movement above a fade collapses, and
you animate only `transform` and `opacity`.

## In-page CTAs should glide, not jump

A loyalty page links to itself constantly: "start earning" to the earn band,
"spend your points" to redeem. A hard jump makes the page feel like eight
disconnected screens.

**Check whether the theme already handles it before adding anything.** Grep for
`scroll-behavior` and `scroll-margin-top`. Two things to look for, and both are
common:

- The rule exists but **the stylesheet is never enqueued**. A `custom.css` full
  of good intentions that no `theme.liquid` references does nothing. Confirm the
  file is actually loaded before crediting it.
- The rule is scoped to something unrelated, for example `scroll-behavior: auto`
  on a slider, which is not a global setting.

When you do add it, **scope it to the loyalty page** rather than setting
`html { scroll-behavior: smooth }` globally: that changes how the merchant's
entire store scrolls, which is not what they asked for.

Drive it from a delegated click handler on the page root, so it covers bands
rendered later:

```js
document.addEventListener('click', function (event) {
  var link = event.target.closest('a[href*="#"]');
  if (!link || !link.closest('.joy-oneup')) return;
  /* ...same-page check, then... */
  event.preventDefault();
  target.scrollIntoView({ behavior: reduce ? 'auto' : 'smooth', block: 'start' });
  history.replaceState(null, '', location.pathname + location.search + hash);
});
```

Three details that matter:

- **`replaceState`, never `location.hash = ...`.** Assigning the hash makes the
  browser jump to the anchor immediately, cancelling the smooth scroll you just
  started, and stacks a history entry so Back returns to the same page.
- **`scroll-margin-top` on the target**, or a sticky header hides the heading you
  just scrolled to. If the theme has an unused rule with its own numbers, use
  those: they are the merchant's measurements of their own header.
- **Honour `prefers-reduced-motion`** by jumping instead.
