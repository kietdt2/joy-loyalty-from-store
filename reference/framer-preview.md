# Framer preview and gate 1

The merchant sees the page before it exists. A published Framer URL they open in
their own browser, approve or mark up, and only then does Liquid get written.

This is the most expensive stage to get wrong, because everything after it
inherits the approved layout.

## Setup, in order

1. Run `npx @framer/agent@latest setup` with Bash and **let it complete**. This
   is a hard precondition of the `framer` skill.
2. If it needs an interactive login, tell the user to run it themselves with the
   `!` prefix in this session so the output lands in the conversation. Do not
   try to authenticate for them.
3. Only then invoke the `framer` skill. Read the task map it generates
   (`projects/<safeProjectId>/index.md`) before touching components.

If Framer cannot be set up at all, say so plainly and offer the fallback: a
local static HTML comp of the same layout, opened in the browser, serving the
same gate. The gate itself is not optional; only its medium is.

## What belongs in Framer, and what does not

Framer is for the surfaces you actually control: the loyalty page and any
theme-side blocks. **The Joy widget is not one of them.**

Recreating the V4 widget in Framer produces a picture of a product that cannot
be built. It reads as a proposal, the merchant approves it, and then it turns
out the widget's layout is fixed by the component library and only colours and
fonts were ever negotiable. Do not put a redrawn widget in the comp.

For the widget, do one of these instead:

- Run the real components from the Joy repo and screenshot or link that.
- Show its current live state and list the settings that would change it.

Either way, label it clearly as the widget's real configuration surface, not a
design.
-> `reference/joy-app-source.md`

## Composing the preview

Build the bands in the order the direction contract fixed, using only the token
sheet values. The preview's job is to answer "is this the right page for us",
so it must be honest about layout, hierarchy, type, colour and rhythm.

- **Desktop and mobile both.** Author the mobile frame; do not let the merchant
  approve a desktop-only design and discover mobile later.
- **Placeholder loyalty data, clearly plausible.** A member with a name, a real
  looking balance with thousands separators, three tiers, four earn actions,
  four redeem options, a referral link. Numbers must look like the merchant's
  own price band, not like `1234`.
- **Draw the states the design has to cover**, at minimum logged-out and empty.
  These are where generic loyalty pages fall apart and the merchant should see
  your answer.
- **Real merchant copy.** Headings and CTAs in the register mined from their
  store, tiers in their vocabulary. Lorem ipsum invalidates the gate.
- **Motion as specified**, including its reduced-motion collapse.

Follow the em-dash ban here too: no U+2014 or U+2013 as a separator anywhere
visible. It carries into the Liquid, and the downstream skill enforces it.

## Publishing and handing over

Publish, capture the URL, and send it with this framing, adapted to the user's
language:

- what the link is: a **visual preview**, not a working loyalty page,
- the data is **placeholder**; real balances arrive with the SDK wiring,
- what you want back: approve, or list changes,
- what happens next: approval starts the Liquid build.

Then include the direction contract inline so they are approving the reasoning
and not only the picture.

## GATE 1

**Stop here. Write nothing into any theme until the user has opened the link and
approved it.**

If they ask for changes: edit in Framer, republish, send the updated link, and
say what changed. Iterate in Framer, which is cheap. Do not argue for version
one, and do not start Liquid "in parallel to save time" - a rejected layout
makes every section file wasted work.

Record the approved state before moving on:

```
APPROVED - <date>
framer url      <...>
approved by     <user>
changes made    <what came back from review, and what you did>
locked          band order, hierarchy, token sheet, copy register
```

That locked line is what `joy-loyalty-build` implements. Deviating from it later
needs the user's word, not your judgement.
