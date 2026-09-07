# Design direction

The store analysis says what the brand *is*. The direction says what the page
*does*. It is a contract: the Framer preview is judged against it, and so is the
finished build.

## Invoke impeccable

Call the `impeccable` skill and let it own the design decisions. It picks the
world, sets the quality bar, and its finish reviewer later grades the build
against exactly this contract. Feed it:

- the store analysis record,
- the locked token sheet,
- the merchant's written requirements verbatim, if any,
- the band candidate list below.

Do not pre-empt its choices. Your job is to give it a merchant-specific brief,
not a finished opinion.

## Choosing the bands

A loyalty page is assembled from a fixed vocabulary of bands. Pick the set and
the order from the merchant's purchase rhythm, not from a template.

| Band | Include when | Data |
|---|---|---|
| Hero / join | always | none, auth gate only |
| Member status | accounts exist | `joy.customer()`, `joy.tiers()` |
| How it works | the mechanic is not obvious in three seconds | none |
| Ways to earn | always | `joy.earnPrograms()` |
| Ways to redeem | always | `joy.redeemPrograms()` |
| VIP tiers | tiers are configured, or the brand rewards frequency | `joy.tiers()` |
| Referral | the buyer shares, gifts or recommends | `joy.generateLinkReferral()` |
| Rewards carousel | catalogue is broad and visual | `joy.rewardList()` |
| FAQ | terms are conditional or the price band is premium | none |
| Closing CTA | always | none, auth gate only |

Order heuristics, applied to the analysis:

- **Replenishment** (coffee, supplements, consumables): earn leads, tiers early,
  referral late. The mechanic is habit.
- **One-off / considered** (furniture, electronics): referral leads after the
  hero, tiers demoted. The mechanic is advocacy, not frequency.
- **Gifting / seasonal** (party, florals, holiday): redeem leads, rewards
  carousel high, tiers as aspiration.
- **Premium / luxury**: tiers lead, points language de-emphasised, fewer bands
  and more air. Density down.
- **Signed-in-first stores** (strong account presence): member status directly
  under the hero.

## The direction contract

Show this to the user with the Framer link. It is what gate 1 approves.

```
DESIGN DIRECTION - <brand> loyalty page

read            <one line: what this page is, in this brand's terms>
world           <as chosen by impeccable>
hierarchy       <what wins the eye, first / second / third>
bands           <ordered list, with a reason per band, and REUSE: <section> where
                 an existing theme section covers it>
imagery         <which real storefront photos, where, and why those>
decor           <the brand mark treatment: sizes, opacities, which bands>
motion          <the theme's easing and durations, verbatim; any deliberate
                 override and why; reduced-motion behaviour>
density         <n/10> variance <n/10>  <- from the token sheet dials
copy register   <register, plus 3 real merchant strings being echoed>
tier naming     <names in the merchant's vocabulary, with their source>

UNIQUENESS CLAIM - three decisions this brand caused
  1. <decision> because <evidence from the analysis>
  2. <decision> because <evidence>
  3. <decision> because <evidence>

STATES THE DESIGN MUST COVER
  logged out / loading / empty / error, per SDK band

ASSUMPTIONS
  <every inference, one per line>
```

An entry in the uniqueness claim whose evidence is "the brand is premium" is
not evidence. It must point at something in the record: a real string, a
measured value, a catalogue fact, an observed device.

## Run the page against an anti-slop floor before you call it designed

A loyalty page has a strong gravitational pull toward the category template, and
the pull is strongest in the places that feel like craft. Check these before the
Framer gate, not after the Liquid ships.

**Eyebrows above headings.** The single most common one, and the hardest to see,
because each individual eyebrow looks like a considered label. Read them as a
column: "Ways to earn" over "Points for the things you already do", "Tiers" over
"Three tiers. One is by invitation." Every one restates its heading. If the
heading needs a label to be understood, rewrite the heading. Ship the settings
with no default so a merchant has to choose to add one back.

**A message that restates its own button.** "You are signed in. Join the
programme to start earning." above a button reading "Join the programme" says one
thing twice and neither well. Supporting copy earns its place by adding what the
button cannot: the cost, the catch, the timing. "Free, and your next order starts
counting." is a different sentence, not a louder one.

**Unicode glyphs for icons.** A star as `&#9733;` renders in whatever the system
font decides, at a weight belonging to no design system, and differently on every
platform. If the rest of the page draws its icons, these are drawn too. And five
outlines with three filled is a rating widget; three filled stars is a rating.

**Identical cards as the page structure.** Earn, redeem, tiers and proof all want
to be a grid of icon-heading-text. When every band is the same card at a
different size, the page has no rhythm and nothing to look at.

**The hero-metric block.** Big number, small label, three across, thin rule.
Correct for a stat band and a cliché everywhere else.

**A single flat type ramp across every band.** Sizing headings by importance is
not typography, it is scale. A page where the only difference between the hero
and the seventh heading is `font-size` has no voice, and asking for "better
typography" and getting a bigger ramp back means the brief was misread.

The invocation is `impeccable`, and the file that carries these is its
`reference/craft-floor.md`. Read it before editing UI, not as a review after.

## When a brief asks for ornament the theme does not have

"Elegant", "luxurious", "calligraphic", "chữ có nét hoa văn" — briefs of this
kind ask for a quality that a geometric sans cannot be coaxed into. Poppins,
Harmonia Sans, Inter, Montserrat have no calligraphic weight anywhere in them.
No amount of tracking, weight or size produces one. The note has to be
introduced, and the first honest step is saying so rather than shipping a larger
ramp and calling it typography.

Introduce it on **a phrase, not a line**. One display serif carrying two or
three words inside an otherwise sans heading is typography; the whole heading in
a second face is a font swap, and it reads as one.

```css
.flourish {
  font-family: 'Playfair Display', Didot, 'Bodoni 72', Georgia, serif;
  font-style: italic;
  font-size: 1.12em;              /* a serif reads smaller at the sans's size */
  letter-spacing: 0.005em;        /* italic serifs tighten on their own */
  font-feature-settings: 'dlig' 1, 'swsh' 1, 'calt' 1;
  padding-inline: 0.06em;         /* the italic overhangs its box both sides */
}
```

The ornament lives in `dlig` and `swsh`, and both are **off by default**. A
display serif without them is just a serif.

Use the face's **true italic**, never `font-style: italic` on the sans. That
synthesises an oblique by shearing the glyphs, which is why faux-italic sans
always looks cheap.

Load it with Shopify's own `font_face`, not a Google Fonts `<link>`: the
storefront never talks to a third party, and Shopify emits the right woff2 and
`unicode-range`. Load **only the style you use**.

```liquid
{%- assign flourish = 'playfair_display_i4' | font_modify: 'style', 'italic' -%}
{{ flourish | font_face: font_display: 'swap' }}
```

**Pick the phrase where the meaning already is.** The emotional word, not a
random noun: *"Three tiers. One is `by invitation`."*, *"They `asked` where you
got them."* Ornament placed on a function word is decoration; placed on the
point of the sentence it is emphasis.

Make it a setting the merchant can edit, and **match on the literal substring**
of the heading rather than on a marker character. The heading setting then stays
plain text — correct in the editor sidebar, in search results, and in the
accessibility tree — and a merchant who never opens the accent field still gets
a valid heading.

Two implementation traps:

- **Liquid `split` drops a leading empty segment**, so `parts[0]` is not
  reliably the text before the accent: when the accent opens the heading,
  `parts[0]` is what follows it. Rebuild the head with `remove_first` instead,
  and test the accent at the start, in the middle, and as the entire heading.
- **A heading written by JS cannot use the Liquid snippet.** Apply the same wrap
  after the write, with `textContent` plus a constructed `<span>` — never
  `innerHTML` with the value interpolated, because a greeting contains a
  customer's own name.

Section files, SDK method wiring, the 750px contract, schema rules and the four
implemented states all belong to `joy-loyalty-build`. Name them in the contract
so the design accounts for them; do not build them here.
