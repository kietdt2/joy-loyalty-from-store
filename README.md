# joy-loyalty-from-store

A Claude Code skill that turns a Shopify store URL into a loyalty page built for
that one merchant — analysed from their storefront, previewed in Framer before
any code exists, and shipped as Liquid sections on an unpublished dev theme.

It is for the Avada Joy loyalty app. It reads the Joy monorepo for the real SDK
contract rather than guessing at method names, and it refuses to redraw the
parts of Joy that a merchant cannot actually change.

## Built with this skill

Two merchants, two loyalty pages, no shared layout. Both previews are the design
proof: they mirror what actually shipped as Liquid on the merchant's own theme.

| Merchant | What they sell | Design proof |
|---|---|---|
| [oneupballoons.ca](https://oneupballoons.ca) | Balloons and party installations, Canada | **[Loyalty page](https://social-guest-117624.framer.app)** · [Widget V4](https://social-guest-117624.framer.app/widget) |
| [chiclara.com](https://chiclara.com) | East Asian designer fashion — women's clothing, bags, shoes | **[Loyalty page](https://attractive-emojis-099797.framer.app)** |

The One Up Party page is the worked example throughout this README: a display
serif carrying one phrase per heading, the merchant's own photography, their
reward names leading every card, and a tier ladder that shows the
invitation-only tier rather than hiding it.

---

## What it produces

| Deliverable | What it is |
|---|---|
| Framer preview URL | The design, before a line of Liquid. Approval gate. |
| Live demo URL | The real page on an unpublished Shopify dev theme. |
| Delivery folder | `[Joy-Loyalty-Team] <Original Theme Name>` — the merchant's theme byte-identical, plus the new sections. |
| Report | Analysis, token sheet, section-to-band map, which band reads which SDK call, merchant setup steps, and every remaining gap stated honestly. |

## Installing

Drop the folder into your skills directory:

```bash
git clone <this-repo> ~/.claude/skills/joy-loyalty-from-store
```

Claude Code picks it up on the next session. Invoke it by pasting a store link
and asking for a loyalty page, or explicitly:

```
/joy-loyalty-from-store https://brand.myshopify.com
```

### What it expects to find

| Dependency | Needed for | If absent |
|---|---|---|
| The Joy monorepo, checked out locally (`/Users/avada/WebstormProjects/joy` by default) | The V4 widget, loyalty-page components, icon registry, SDK contract | Falls back to bundled docs and flags V4 specifics as unverified |
| `impeccable` skill | Design direction and the finish review | Ask for it; the quality floor lives there |
| `framer` skill + `npx @framer/agent@latest setup` | The preview at gate 1 | No preview, no gate |
| `joy-loyalty-build` skill | Section authoring, schema rules, the responsive contract | This skill is only the upstream half |
| Shopify CLI, authenticated | The dev theme and the live demo URL | Zip-and-upload fallback, demo URL pending |

## Inputs

| Input | Required | If missing |
|---|---|---|
| Shopify store URL | **yes** | Stop and ask. Everything keys off it. |
| Written requirements | no | Infer, and list every inference in the direction contract so the merchant can correct it |
| Exported theme folder | no | Analyse the live storefront, and say the palette came from the rendered page |
| The shop's Joy data blob | no, but **ask** | Without it you are guessing tier names, thresholds, programmes and reward values — it changes the design, not just the copy |

## The workflow

Two gates. It stops dead at each.

```
0  Open the store        shopify theme open --store <store>
1  Analyse               products, price posture, palette, type, voice
2  Extract               tokens, real photography, reusable sections, motion
3  Direction             impeccable picks the world and sets the bar
4  Framer preview        ── GATE 1 ── the merchant opens the link and says yes
5  Build                 joy-loyalty-build converts it to Liquid ── GATE 2 ──
   + surfaces            header points card, cart redemption block
6  Delivery folder       original untouched, additions alongside
7  Dev theme             unpublished push, capture the preview URL
8  Verify                three widths, five states, finish review
9  Report
```

An interactive version of this workflow, with the two gates and the places a
build silently stalls, is in [`docs/workflow.html`](docs/workflow.html). Open the
file in a browser: it is self contained, needs no server, and carries its own
dark mode, search and export.

Step 0 is one command and it settles four things that are expensive to discover
after a build: whether the CLI is authenticated **against this shop**, whether
the handle resolves, what the live theme is actually called, and whether a
previous Joy delivery is already sitting there.

## Design rules it enforces

**Uniqueness is a stated claim, not a vibe.** Every build carries at least three
brand-specific decisions traceable to the analysis, named in the report. A candy
brand and a workwear brand must not come back with the same page in different
colours. Recolouring a layout does not count.

**The merchant's own photography, or it has failed.** A theme export ships UI
icons, not brand imagery — the photographs live in Shopify Files and have to be
pulled from the live storefront. A loyalty page with no photography on it has
almost certainly failed the uniqueness rule.

**Reuse their sections before authoring new ones.** Anything non-standard in
`sections/` was built for this merchant, matches their design language by
definition, and usually has a schema the merchant already knows how to edit.

**Copy their motion, not your taste.** The theme's easing and durations,
verbatim. Override the character of a hover if the direction calls for it, and
say so out loud.

**Never publish. Never copy their theme wholesale.** Dev themes and preview
links only; publishing is the merchant's own act.

## What is designable, and what is not

A "loyalty page" brief usually means four surfaces, and they are **not** equally
yours:

| Surface | Who owns the design | What you can change |
|---|---|---|
| Loyalty page | you, in theme code | everything |
| Header points card | you, in theme code | everything |
| Cart redemption block | you, replacing the app block | everything |
| **Widget V4** | **the Joy component library** | only the values in the settings-to-token table |

Widget V4 is not a design you recreate — it is a shipped component library you
configure. Drawing a lookalike in Framer and presenting it as a proposal is a
fabrication. Run the real components, or show the current configuration and name
the settings that would change it.

## Reference files

`SKILL.md` is the workflow. The detail lives in `reference/`, loaded as needed.

| File | Read it for |
|---|---|
| `joy-app-source.md` | Joy monorepo map, settings-to-CSS-token contract, running the real V4 widget, the icon registry, plan gating, widget deep links |
| `states-and-auth.md` | The five states, the three audiences (signed out / signed in but not a member / member), skeletons, and the theme-editor re-render trap |
| `store-analysis.md` | What to read off a storefront, in what order |
| `brand-extraction.md` | Token extraction, and which source wins |
| `theme-assets-and-motion.md` | Fetching real photography, floating brand marks, finding reusable sections, copying motion tokens, in-page scrolling |
| `design-direction.md` | Invoking impeccable, the direction contract, the anti-slop floor, and introducing ornament a geometric theme cannot supply |
| `surfaces-beyond-the-page.md` | The header points card in both placements, replacing the app's redemption block, getting a code into the cart |
| `framer-preview.md` | Composing and publishing the preview, running gate 1 |
| `handoff-to-build.md` | The exact payload handed to joy-loyalty-build |
| `delivery-folder.md` | The folder contract, naming, template naming |
| `dev-theme-demo.md` | CLI auth, `theme open`, the unpublished push, the pull-edit-push loop |
| `verification.md` | The width and state matrix, the finish review, the report format |

### Vendored from joy-loyalty-build

The downstream skill owns section authoring, and these three files are the parts
you need while designing rather than after. They are copies, kept here so a
reader of this repo does not have to have the other skill installed to
understand what the handoff assumes.

| File | Read it for |
|---|---|
| `build/sdk-wiring.md` | Which band reads which SDK call, the four states, rendering a list from the SDK rather than from blocks, widget deep-link hashes, and the tier-progress traps |
| `build/page-assembly.md` | The page template contract, reusing the theme's own product card, and the three ways a setting is silently dropped on push |
| `build/sdk-api.md` | The `window.joyInstance` method reference, page keys with their deep-link hashes, and plan gating |

## Hard-won details

Things the skill encodes because getting them wrong ships a page that looks
finished and is not.

**Opening the widget is `window.avadaJoyTrigger()`.** Not a `joy:open` event,
nothing in Joy listens for one. Not `joyInstance.openWidget()`, because the plan
proxy intercepts it on a gated shop and returns `undefined`.

**Deep link with a URL hash, and prefer it to scripting.** The widget installs
its own `hashchange` listener and routes on the literal hash
(`FloatingButton.js:174-230`), so a plain `<a href="#joy-ways-to-earn">` is the
whole integration: no script, and it cannot lose the race where `openPage()` is
still undefined because the widget's React tree has not mounted. The seven
hashes are defined in `const/deeplinkHash.js`:

```
#joy-home  #joy-loyalty  #joy-ways-to-earn  #joy-ways-to-redeem
#joy-referral-program  #joy-activities  #joy-rewards
```

The names are not all guessable: it is `#joy-referral-program`, not
`#joy-referral`. Anything outside that list falls through the router's switch
and does nothing. `FloatingButton.js:754-758` deliberately skips resetting the
widget to its initial page while a hash is present, which is what stops the deep
link being overwritten by the default screen.

**Signed in ≠ a member.** With opt-in enrollment, a customer can be signed in to
Shopify and still have `type: 'guest'` in Joy. Three audiences, not two. Showing
the middle group a member view gives them a zero balance and no way to fix it.

**Plan gating returns stubs, not errors.** With `canUseJoySDK: false`,
`joyInstance` is a Proxy: `customer()` resolves `null`, `redeem*` resolve
`{success: false, message: 'Ultimate plan required'}`. A section that "gracefully
handles a missing SDK" still renders empty.

**Redeeming is `joyInstance.redeem(id, points)`.** `redeemProgram` belongs to the
older `window.JoyJs`, a different object. Two failure shapes have to be caught:
the SDK's `{status: false, error}` and the proxy's `{success: false, message}`.

**Tier progress is `tierPoint`, and the thresholds may not be points at all.**
`customer.point` is the spendable balance and falls when a member redeems, so
driving a tier bar from it demotes people for using the programme. Use
`tierPoint`. Separately, `tierSettings.entryMethod` decides what the thresholds
measure, and `targetPoint` is expressed in *those* units in spite of its name:
on a `moneySpent` shop `targetPoint: 250` means $250, not 250 points. Joy
accumulates the qualifying amount into `tierPoint` itself, so the comparison
stays `tierPoint >= targetPoint` on every entry method and only the wording
changes.

**A progress bar and its own scale have to measure the same span.** Filling the
bar across the current segment while labelling the axis with the whole ladder
renders a member who has just reached a tier at zero percent, sitting under a
label that says they are a third of the way along.

**A push can report success and still not apply.** A Liquid error in one file
prints an error block, the command reports "pushed with errors", and every other
file in the batch lands while that one does not. Settings that fail schema
validation are dropped silently and the push still succeeds, so a product
setting written as a bare handle rather than `shopify://products/<id>` becomes
`null`. Verify by pulling back what landed, never by trusting the message.

**Pull before every edit, not before every push.** The merchant edits the same
theme in admin while you work, and `theme push` is last writer wins. Pulling at
push time is already too late, because the change was authored against a stale
base.

**Applying a code to the cart is `GET /discount/<code>`.** There is no cart.js
field for discount codes — posting `discount` to `/cart/update.js` returns 200
and changes nothing.

**The theme editor is a sixth state.** Changing a setting replaces the section's
DOM and every JS-rendered node dies. Without a `shopify:section:load` handler the
merchant meets a section that empties itself and only comes back on reload.

**Push scoped, and pull the whole theme first.** The remote theme changes for
reasons unrelated to you — a merchant edits a setting, an app renames its handle
and Shopify rewrites every block reference. A bare `theme push` from a stale
folder destroys all of it.

**`{{ }}` is Liquid output syntax.** Copy placeholders use `[name]`, `[points]`,
`[tier]`. `{{name}}` in a setting default is rejected outright.

## Licence and provenance

Written for the Avada Joy loyalty team. The Joy SDK contracts documented here
were read from the Joy monorepo, not from public documentation, and are accurate
as of September 2026 — verify against the source before relying on them.
