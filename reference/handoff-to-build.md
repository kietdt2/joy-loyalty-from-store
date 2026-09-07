# Handoff to joy-loyalty-build

Approval turns the design into an input. From here `joy-loyalty-build` owns the
implementation and you stop making implementation decisions.

## What each skill owns

| Owned by **this** skill | Owned by **joy-loyalty-build** |
|---|---|
| Store analysis, token sheet, direction contract | Section files, architecture choice per band |
| Framer preview and gate 1 | SDK wiring, boot gating, the four states |
| Band cut list and order, with reasons | 750px responsive contract, mobile build |
| Delivery folder, dev-theme push, final report | Schema rules, naming, page template assembly |

Do not reimplement anything in the right column. Do not let the build skill
re-decide anything in the left column without asking the user.

## The handoff payload

Invoke `joy-loyalty-build` and hand it exactly this. It expects a design as
input; the approved Framer layout is that design.

```
INPUT STYLE      section-split, authored from Framer (not a Figma link)

DESIGN SOURCE
  framer url     <published preview>
  desktop width  <px, as composed>
  mobile frame   authored at 390, see <url or frame name>
  screenshots    <scratchpad paths, per band, both widths>

TOKEN SHEET      <the locked sheet, verbatim>
                 ship as snippets/joy-<prefix>-tokens.liquid from the topmost section

TARGET THEME     <path to the delivery folder>
PREFIX           joy-<brand-kebab>          <- one prefix for the whole page family

CUT LIST         one row per band, in flow order
  # | band | file joy-<prefix>-<name>.liquid OR reuse <existing section> |
      archetype | anchor id | mobile behaviour

  ...

REUSED SECTIONS  theme sections to configure rather than rebuild
  <section file> -> <band>, driven by <the settings/blocks it already exposes>

ASSETS           real files, already fetched, with their source URLs
  <local path> <- <storefront URL>   used in <band>
  brand mark    <path>   floating decor: sizes, opacities and bands

MOTION           the theme's own values, to be emitted in the sections
  easing     <cubic-bezier from base.css>
  durations  <the token values actually used>
  reveal     <transform + opacity + duration>
  overrides  <any deliberate deviation, with its reason>

COPY             <headings, CTAs, tier names, per band, final and approved>
STATES           <what the design specified for logged out / loading / empty / error>
JOY INSTALLED    <yes | no | unknown>
CONSTRAINTS      <merchant requirements verbatim, plus font substitutions made>
```

Because the design is authored rather than read from Figma, **skip the Figma MCP
extraction stage** of that skill and enter at its segmentation step with the cut
list already filled. Say so explicitly when you invoke it, or it will look for a
Figma link and stall.

## Its own gate

`joy-loyalty-build` will present its cut list and require an explicit yes before
writing files. That is gate 2. Honour it. Your gate-1 approval does not stand in
for it: the user approved a look, not a file plan.

If it proposes an architecture that contradicts the approved design, the design
wins on layout and hierarchy, and the build skill wins on data mechanics. When
those genuinely collide, put the collision to the user rather than resolving it
silently.

## Sanity checks before you invoke

- Every band in the cut list appears in the direction contract, and vice versa.
- Every SDK-driven band names a method that exists in that skill's
  `resources/joy-sdk/`. Never invent a method name.
- The prefix is unique to this merchant, so two Joy pages on one store cannot
  collide on a custom element name.
- The target path is the **delivery folder**, never the merchant's live theme
  directory.
