# Segmenting the frame and assembling the page

## 1. Figma band to Joy archetype

Walk the root frame top to bottom. Each top-level child that spans the full width
is one **band** = one section. Match it against this table, then build it with
the `joy-section` architecture named in the third column.

| Band looks like | Joy section | Architecture / template (`joy-section`) | Data |
|---|---|---|---|
| Big headline, subcopy, one or two CTAs, art or product shot | `joy-hero-banner` | liquid-static | none, plus auth-gated CTA |
| Numbered or icon steps, "how it works", 3 to 4 items | `joy-how-it-works` | liquid-static, `step` blocks | none |
| Points balance card, member greeting, tier badge, progress bar | `joy-points-balance` | SDK-driven | `joy.customer()` + tiers |
| Grid of earning actions with icons and point values | `joy-ways-to-earn` | `ways-to-earn-section.liquid` + `ways-to-earn-modals.md` | `joy.earnPrograms()` |
| Grid of rewards with a point cost and a redeem button | `joy-ways-to-redeem` | `ways-to-redeem-section.liquid` + `ways-to-redeem-modals.md` | `joy.redeemPrograms()` + `joy.redeem()` |
| Tier ladder, columns or cards, perks per tier, "you are here" | `joy-vip-tiers` | `vip-tiers-section.liquid` | `joy.tiers()` + `joy.customer()` |
| Referral link, copy button, share icons, invite stats | `joy-referral` | `referral-section.liquid` | `joy.generateLinkReferral()` |
| Accordion of questions | `joy-faq` | liquid-static, `question` blocks | none |
| Banner strip with a single CTA near the page end | `joy-cta-banner` | liquid-static | auth-gated CTA |
| Anything else visual | `joy-<kebab-of-what-it-is>` | liquid-static | none |

Judgement calls:

- **A band that mixes archetypes** (balance card sitting inside the hero) stays
  one section only if the two parts always move together. If the merchant would
  reorder them independently, split it.
- **Repeated cards are blocks, not hard-coded markup**, whenever the merchant
  would plausibly edit them (steps, FAQ, perks). Program-driven grids stay
  SDK-driven and are never blocks.
- **A band showing mock loyalty numbers is SDK-driven**, even if the design draws
  it as flat text. Mock values in Figma are placeholders, not content.
- **Do not skip a band you cannot classify.** Build it as liquid-static and name
  it for what it shows.

Show the cut list to the user before building anything:

```
Figma node 12:340  -> joy-hero-banner        liquid-static
Figma node 12:512  -> joy-how-it-works       liquid-static, 3 step blocks
Figma node 12:788  -> joy-ways-to-earn       SDK, joy.earnPrograms()
...
```

## 2. The page template

Real shape, as used across the themes in `/Users/avada/Documents/joy`:

```json
{
  "sections": {
    "hero":           { "type": "joy-hero-banner",   "settings": { ... } },
    "how_it_works":   { "type": "joy-how-it-works",
                        "blocks": { "step_1": { "type": "step", "settings": { ... } } },
                        "block_order": ["step_1", "step_2", "step_3"],
                        "settings": { ... } },
    "ways_to_earn":   { "type": "joy-ways-to-earn",  "settings": { ... } }
  },
  "order": ["hero", "how_it_works", "ways_to_earn"]
}
```

- Path: `<theme>/templates/page.joy-loyalty-page.json`. **Always this exact
  name.** Do not name the file after the merchant, the brand, or the shop's
  `handleLoyaltyPage` value: `page.chiclara-loyalty-page.json` is wrong even when
  the shop's loyalty page handle is `chiclara-loyalty-page`. The part after
  `page.` is a **template suffix**, an internal identifier the merchant picks
  from the Theme template dropdown, and it is unrelated to the page's URL handle.
  Naming it after the shop makes the template unrecognisable in a theme that
  serves more than one merchant, and it hides the fact that this is the Joy
  loyalty template. Use a versioned suffix (`page.joy-loyalty-page-v2.json`) only
  when the theme already ships one, and say which file you wrote.
- Section keys are semantic snake_case (`ways_to_earn`), not the section type.
- `order` is the Figma top-to-bottom order. Every key in `sections` appears in it.
- Carry the measured values into `settings` (colors, sizes, radii, columns,
  paddings) so the page renders correctly on a fresh install without the merchant
  touching the customizer.
- Blocks need both the `blocks` map and `block_order`.
- **A suffix template renders nowhere until a page is assigned to it, and that
  assignment is not a theme file.** This is the single most common reason a
  correct build appears to have done nothing. `templates/page.joy-loyalty-page.json`
  sits inert in the theme; the page keeps rendering `templates/page.json` until
  someone opens Shopify admin, Content > Pages, and sets Theme template to
  `joy-loyalty-page`. There is no theme file, no `?view=` parameter and no push
  flag that does this: `?view=` works for alternate *Liquid* templates, not JSON
  page templates. The Theme CLI cannot do it either, because creating a page and
  setting its `template_suffix` needs Admin API `write_content` scope while the
  CLI holds only theme file scope.

  Consequences to plan for, not discover:

  - **Budget for it before verification.** Step 8's screenshots are impossible
    until the assignment exists. Either get the merchant to assign it first, or
    obtain an Admin API token, or state plainly that the build is verified by
    schema and preflight only and the visual pass is pending.
  - **Do not fake it by overwriting `templates/page.json`.** It renders, and it
    is tempting, but `page.json` is the default template for *every* page on the
    theme, so the About and Contact pages become the loyalty page too. If it is
    used at all as a momentary render check, back the file up first and restore
    it in the same turn.
  - **Check whether the loyalty page already exists.** `AVADA_JOY.shop.handleLoyaltyPage`
    names the page Joy is already pointed at, usually built from Joy app blocks
    in `page.json`. Reassigning that page to the new template replaces the app
    block page, which is normally the intent, but it is the merchant's call and
    it changes what their customers see. Ask before assuming.
  - **Render the theme's own product card, never a hand built one.** Any band
    that shows products (rewards, earn examples, a tier's featured item) uses
    `{%- render 'product-item' -%}`, or whatever the theme calls its card
    snippet: check what `main-collection.liquid` and `featured-collections.liquid`
    render. It inherits quick view, the add-to-cart drawer, variant swatches,
    vendor, sale and sold-out labels and colour counts for free, and it keeps
    tracking the theme instead of drifting from it. A hand built card looks
    similar on day one and behaves worse forever.

    Four things that follow from it:

    - **Blocks take a `product` picker, not image URLs and text.** Prices,
      availability and images then stay live. A hardcoded CDN URL is stale the
      day the merchant reshoots.
    - **`show_cta` moves the button.** With `show_cta: true` the snippet appends
      a CTA below the card; omit it and the quick action renders inside the
      image wrapper instead. Read the snippet's conditionals before choosing:
      the in-image branch is usually gated on `show_cta != true`.
    - **Overlay your own badge, never edit the snippet.** Put the points stamp
      on the wrapping `<li>` with `position: relative`, and give the stamp
      `pointer-events: none` so it cannot swallow a tap meant for the quick view
      button underneath it.
    - **Hide the theme's sale label if it collides.** `SAVE $X` usually occupies
      the same top-left corner, and a discount badge beside a points badge asks
      the reader to compare two unrelated numbers. Scope the override to the
      section so the rest of the store keeps its badges.

    Revealing quick view on hover: gate it behind `@media (any-hover: hover)`.
    On touch there is no hover state, so an ungated rule makes the button
    permanently unreachable. Animate `opacity`, not `display`, and include
    `:focus-within` so keyboard users can reach it.

  - **Pull before every edit, not before every push.** The merchant edits the
    same theme in the Shopify admin while you work, and the theme editor writes
    straight to the live theme files. Editing a local copy you pulled twenty
    minutes ago and then pushing silently reverts whatever they changed in the
    meantime, with no conflict and no warning: `shopify theme push` is
    last-writer-wins. Pulling at push time is already too late, because you
    authored the change against a stale base.

    The habit that works:

    ```bash
    shopify theme pull --store <shop> --theme <id> --path /tmp/remote-now
    diff /tmp/remote-now/sections/joy-x.liquid <local>/sections/joy-x.liquid
    ```

    Diff before touching a file. If it differs, adopt the remote first and
    re-apply your change on top; never assume your copy is authoritative just
    because you wrote it. Escaping a near miss because you happened to edit a
    different file than they did is luck, not method.

    Also expect two benign diffs that are not merchant edits: Shopify prepends
    an auto-generated header comment to JSON templates it has touched, and it
    reorders setting keys. Adopt those silently rather than reverting them.

  - **Settings that fail schema validation are silently dropped, not rejected.**
    Shopify validates every value in a JSON template against the section's
    schema **as it exists on the theme**, discards whatever does not fit, and
    still reports the push as successful. Three ways this bites:

    - **A `product` setting needs `shopify://products/<numeric-id>`.** A bare
      handle is accepted by the push and stored as `null`. Get ids from
      `/products.json`. Values written through the theme editor may come back
      as handles; that form is fine once Shopify itself wrote it, so preserve
      what you pull rather than normalising it.
    - **A block `type` must not collide with one of its own setting ids.** A
      block typed `product` holding a setting id `product` silently loses the
      value. Name the type for the role (`reward`, `earn_item`, `tier`).
    - **Push the section before the template.** New settings only validate once
      the schema that defines them is on the theme. Pushing both in one command
      is not enough when the section fails to compile, because the template is
      then checked against the old schema and every new value is stripped.

    Always verify by pulling the template back and reading the values, not by
    trusting the success message.

  - **A push can report success and still not apply.** A Liquid syntax error in
    one file prints an error block but the command exits reporting the theme was
    "pushed with errors", which is easy to skim past, and every other file in the
    batch lands while that one does not. Grep the output for `error`, and when a
    section carries new schema, pull it back and confirm the settings arrived.
    One real example: `{% for tab in 'a,b' | split: ',' %}` is invalid because
    Liquid's `for` cannot take a filter inline. Assign first, then loop.

  - **A stray pushed template cannot be removed with `--only`.** `shopify theme
    push --only <file>` uploads but never deletes, `--nodelete=false` does not
    help, and `shopify theme delete` removes whole themes rather than files. To
    delete one remote file: pull the full theme to a temp directory, delete the
    file there, then push the whole tree so the sync removes it.
- JSON templates take `/* */` comments in Shopify, but keep them out unless the
  theme already uses them.

## 3. Per-section checklist before moving on

- Section id used to scope every CSS rule.
- Token snippet rendered, every custom property carrying a fallback.
- Schema: no blank `default`, `padding_top`/`padding_bottom` pair, a `presets`
  entry, headers grouping the settings, `disabled_on` header/footer.
- SDK sections: `joy:ready` plus already-mounted check plus timeout fallback, and
  the loading / logged-out / empty / error states actually built.
- Auth-gated CTAs hidden for signed-in customers via the Shopify signal.
- Assets copied into `<theme>/assets/` with `joy-` names and referenced by
  `asset_url`.
- Pre-flight from `taste-design.md §12` run, em-dash scan included.
