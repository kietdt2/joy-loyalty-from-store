# The three surfaces that are not the loyalty page

A "loyalty page" brief almost always includes a header points card and an
in-cart or on-page redemption block. They are smaller than the page and they
fail in ways the page does not, because they live inside theme code the merchant
did not write and inside an app block you do not own.

## 1. The header points card

Joy documents this at `help.joy.so/integrations/theme-integration/`. Read the
doc — the metafield names and the digit-grouping loop are load-bearing — and
then improve on the snippet, because as published it ships three defects.

### What the doc is right about

**The metafields.** `customer.metafields.avada_joy.point` and
`customer.metafields.avada_joy.vipTier.tierName`. Server-rendered, no request,
no SDK, no flash of an empty balance. This is the correct data source for a
header: an SDK read would paint the number after the header has already
composited.

**The digit walk.** Liquid has no number-format filter, so the thousands
separator is built by hand:

```liquid
{%- assign aj_point = customer.metafields.avada_joy.point | split: '' -%}
{%- for digit in aj_point -%}
  {%- assign threeFromEnd = aj_point.size | minus: forloop.index -%}
  {%- if threeFromEnd == 2 and forloop.index != 1 -%}{{ digit | prepend: ',' }}{%- else -%}{{ digit }}{%- endif -%}
{%- endfor -%}
```

The doc warns not to reformat this, and it means it: the loop depends on
`forloop.index` against `size`, and a prettifier that rewrites the whitespace
inside the `if` will emit stray spaces between digits. Capture it and `strip`
rather than editing it.

**The trigger.** `avadaJoyTrigger()`, guarded, because the widget script may not
have booted when a customer clicks early.
-> `reference/joy-app-source.md`

### What to fix before shipping it

**It is a `<div onclick>`.** It opens a panel, sits in a header where every
neighbour is focusable, and cannot be reached by keyboard or announced. Ship a
`<button type="button">`. This costs nothing and is the difference between a
control and a decoration.

**It puts colour inline.** Inline `color: #ffffff` is correct on a dark header
and wrong the moment the merchant switches the colour scheme, and it cannot be
retheme d without editing markup in two files. Put it in a class either way.

Then choose deliberately between two treatments, because they say different
things:

- **Derived from the header** — everything from `currentColor`, e.g.
  `background: color-mix(in srgb, currentColor 7%, transparent)`. One rule reads
  correctly on a light header and a dark one, and the chip recedes into the bar.
  Add an `@supports not (background: color-mix(...))` fallback.
- **A solid brand fill** — the theme's own accent from
  `color_schemes.*.button`, with white type. The chip becomes the only saturated
  element in the bar, which is right when the balance is meant to be *noticed*:
  it is the one thing in the header that belongs to this customer.

Expose the fill as custom properties (`--joy-points-bg`,
`--joy-points-bg-hover`) so a retheme is two values.

On a solid fill, **darken on hover rather than lighten** — the chip is already
the brightest thing in a dark header — and put the focus ring *outside* the
chip, because white-on-magenta disappears.

**Icons on a saturated fill are filled, not outlined.** An outlined star at 18px
on a colour field turns to mush: the counters close and it reads as a hairline
sketch at exactly the size it is used.

**It states the tier as a word.** "1,240 Points | Gold tier" is three nouns in a
row in a 40px-tall bar. The mark already says what the number is; the tier can be
a small uppercase label. Put the full sentence in `visually-hidden` text where it
belongs.

### Two placements, one snippet, and the scope trap

Desktop header and mobile drawer are the same control at two sizes:

| File | Where | Variant |
|---|---|---|
| `sections/header.liquid` | immediately before the account icon `<a>` | `header` |
| `snippets/header-drawer.liquid` | first child of `.menu-drawer__utility-links` | `drawer` |

**Do not put the stylesheet inside the snippet.** `render` gives each include an
isolated scope, so a `{%- unless joy_points_css_rendered -%}` guard cannot see
the assign made by the other include and the CSS ships twice. Split it:

```liquid
{% render 'joy-points-card-styles' %}   {# once, from layout/theme.liquid #}
{% render 'joy-points-card', variant: 'header' %}
{% render 'joy-points-card', variant: 'drawer' %}
```

Hide each variant at the breakpoint where the header collapses to the drawer —
the same one the theme's own account icon uses — so the two never both appear.

Anchor the insertion on markup that occurs exactly once and assert it:

```python
assert t.count(anchor) == 1
```

A header has several account links (mobile, desktop, drawer). Replacing the
first match without checking is how the card lands in the wrong one.

## 2. Replacing the app's redemption block

The Joy app block renders every reward as a card, then puts "You need 500 more
Points to redeem this reward" inside each one and greys the button. For a
customer with a low balance that is six identical rejections in a grid: the
information is correct and the hierarchy is inverted, because what they *can*
have is buried under what they cannot.

A custom block earns its place by splitting the list:

- **Affordable rewards** lead, as cards, actionable.
- **Everything else** collapses to one quiet ladder underneath, each rung
  stating the gap once as a number. Cheapest first in both groups — nearest
  goal, lowest commitment.

State the balance once, between the heading and the rewards, rather than
implying it six times in six rejection messages.

### The SDK contract, verified

From `packages/scripttag/src/Joy.js`, not guessed:

```js
joyInstance.redeem(programId, points)   // -> the /redeem payload
                                        // -> or {status: false, error: '...'}
// success carries result.discount.code
// Joy also fires a window 'joy:redeemCoupon' CustomEvent
```

Note it is **`redeem`**, not `redeemProgram`. `redeemProgram` is the older
`window.JoyJs` SDK in `packages/scripttag/sdk/`, which is a different object from
`window.joyInstance`. Both exist in the repo; only one is on the storefront.

**Two failure shapes have to be caught, not one.** The SDK returns
`{status: false, error}`; the plan proxy returns
`{success: false, message: 'Ultimate plan required'}`. Checking only the first
makes a gated shop show a success state with no code in it.

```js
var failed = !result || result.status === false || result.success === false;
var code = result && result.discount && result.discount.code;
if (failed || !code) { /* restore the button, surface the message */ }
```

**Re-read the balance after a redemption; do not subtract locally.** Joy is the
authority and a tier bonus can move it further than the cost.

**Show the code in the card that produced it.** The customer is looking at that
card. A banner elsewhere on the page makes them hunt for the thing they just
paid for.

**`clipboard.writeText` needs a secure context.** A merchant previewing over
http gets a copy button that silently does nothing. Fall back to a hidden
textarea and `execCommand('copy')`.

### Getting a redeemed code into the cart

Redeeming and applying are **two steps**, and only the first belongs to Joy:

```js
const r = await joyInstance.redeem(programId);   // spends the points
const code = r.discount.code;
await fetch(base + '/discount/' + encodeURIComponent(code));  // attaches it
window.location.reload();                        // totals are theme-rendered
```

`GET /discount/<code>` is the storefront route that attaches a code to the
session cart, and it is what Joy's own `applyDiscountCode.js` calls. **There is
no cart.js field for discount codes** — posting `discount` to `/cart/update.js`
returns 200 and changes nothing, which is the trap here because it looks like it
worked.

Reload after attaching. A discount changes the line totals, the subtotal and any
drawer, all rendered by theme sections you do not control.

**The points are spent before the code exists.** If the attach fails the
customer has paid and has nothing in the cart, so show the code as copyable text
regardless of whether the attach succeeded. Do not reload it off the screen on
the failure path.

### Check the order minimum before spending

A reward with `orderReq: 'min_amount'` will not apply to a cart below
`orderReqAmount`, and the app block lets the redemption go through anyway — the
points are gone and the discount silently fails at checkout. Compare before
offering the button.

`orderReqAmount` is in the shop's major unit and `cart.total_price` is in cents.
Comparing them directly is an off-by-100 that makes every reward look available.

```js
if (cartTotal < Number(p.orderReqAmount) * 100) { /* not yet */ }
```

### Cart styling is cart styling, not loyalty-page styling

A cart page is flat: hairlines on the page background, no cards, no fills, the
theme's money type carrying the numbers. A band lifted from the loyalty page —
dark surface, rounded cards, brand mark — reads as an advert wedged between the
line items, which is exactly what the app block being replaced looks like.

- Rules at the theme's own weight and colour, e.g.
  `0.1rem solid rgba(var(--color-foreground), 0.08)`.
- The balance at the same size as `.totals__total-value`, so it reads as a
  figure in the same column of numbers as the cart total.
- **The theme's own button class**, not the loyalty page's. This control sits
  inches from Checkout, and two button languages in one column is precisely what
  makes an app block look bolted on.
- One row per reward, not a grid of cards. Cards compete with the products
  above them.
- **Actionable rewards first.** This block gets a second of attention on the way
  to Checkout, not a browse.
- Render nothing at all for an empty cart. A rewards prompt with no items is an
  advert.

### Keep one failure local

A redemption that fails should put its message on its own row. Swapping the
whole block to an error state hides every other reward because one of them
failed.

## 3. Keep the vocabulary in step across surfaces

If the ways-to-redeem band calls a reward "Party Fund · 1,000 Points · $12 off",
the redemption block must call it the same thing. Two namings for one reward
reads as two programmes.

Share the naming and formatting helpers rather than reimplementing them per
section, and share the term-list markup so the two bands read as one system.

**Lead with the merchant's name for a reward, never with its value.** "Party
Fund", "Celebration Credit", "Big Day Credit" are what makes the band belong to
this store. Promoting "$25 off" to the heading and burying the name mid-sentence
throws that away and leaves a row of cards that differ only by a number — which
is the generic loyalty page the uniqueness rule exists to prevent.

Drop a value that the merchant already put in the name. "10% Off Your Order"
followed by "10% off" says one thing twice.

## Grid width is a design decision, not a default

`repeat(auto-fit, minmax(Xrem, 1fr))` with a small X silently produces five
narrow columns when a merchant configures seven rewards, and every card title
wraps. Cap it:

```liquid
{%- assign gaps = settings.max_columns | minus: 1 | times: 2 -%}
max-width: {{ settings.max_columns | times: settings.min_card_width | plus: gaps }}rem;
margin-inline: auto;
```

Derive the cap from the gap count rather than adding a fudge factor, or a fourth
card squeezes in. `auto-fit` still handles the collapse down to mobile.

Set the minimum from the longest real reward name in the merchant's data, not
from a round number. Two-word names need roughly 28rem to stay on one line.
