# The Joy app source, and why you read it first

There is a Joy monorepo on this machine. When it is present, it outranks every
other description of how the widget or the loyalty page looks, including the
bundled `joy-sdk` docs in `joy-loyalty-build`, which document the **older
scripttag widget**, not V4.

```
/Users/avada/WebstormProjects/joy      # the team's usual checkout
$JOY_REPO                              # or set this to your own
```

Substitute your own checkout path if it differs; the layout below is what
matters, not the prefix. Confirm it exists before relying on anything here. If it is absent, say so and
fall back to `joy-loyalty-build/resources/joy-sdk/`, flagging that V4 specifics
are unverified.

## The one thing to understand

**Widget V4 is not a design you recreate. It is a shipped component library you
configure.** A merchant cannot move its parts, restructure its layout, or
restyle it freely. They change values in the Joy admin, those values become CSS
custom properties, and the components read them.

So "design the widget" is the wrong frame. The right questions are: which
settings produce the look the merchant wants, and what does the widget actually
look like once those settings are applied.

Drawing a lookalike in Framer and calling it the widget is a fabrication. Do not
do it. Run the real components, or show the merchant's current configuration
honestly and say the widget is fixed.

## Where things live

| Path (under the repo root) | What it is |
|---|---|
| `packages/web-components/src/components/widget/` | The V4 widget: `joy-loyalty-widget.ts` plus ~55 sibling components |
| `packages/web-components/src/components/widget/joy-loyalty-widget.styles.ts` | The widget's real CSS, ~1500 lines, including all the variant and breakpoint rules |
| `packages/web-components/src/components/loyalty-page/` | **Loyalty page components**: `joy-loyalty-page`, `joy-vip-tiers-table`, `joy-referral`, `joy-faq`, `joy-how-it-works`, `joy-my-reward`, `joy-redeem-discount`, `joy-redeem-products`, `joy-rewards-activity`, `joy-loyalty-hero`, `joy-inline-banner` |
| `packages/web-components/src/components/redeem-cart/joy-redeem-cart.ts` | The in-cart redemption block |
| `packages/web-components/src/components/base/` | Primitives: button, card, icon, tabs, modal, badge, progress, and ~30 more |
| `packages/web-components/src/icons/index.ts` | **The icon registry.** Outlined stroke SVGs, ~100 names |
| `packages/web-components/src/styles/tokens.css` | The design token source of truth |
| `packages/scripttag/v4-adapters/helpers/applyBrandingTheme.js` | **Merchant settings to CSS token mapping.** The single most useful file in the repo for this skill |
| `packages/web-components/index.html` | A working demo of every component with real attribute examples |
| `docs/features/widget-v4/` | Design system, migration, typography and redesign notes |

Read `applyBrandingTheme.js` and `index.html` before anything else. Between them
you get the full configurable surface and a copyable usage example.

## The settings-to-token contract

From `applyBrandingTheme.js`. This is what a merchant can actually change:

| Setting (V4 nested, V3 flat fallback) | CSS tokens it sets |
|---|---|
| `theme.primaryButtonColor` | `--joy-primary`, `--joy-primary-hover`, `--joy-primary-light`, `--joy-accent`, `--joy-accent-bg`, `--joy-primary-tint`, `--joy-border-accent`, `--joy-landing-accent*`, `--joy-text-inverse` |
| `theme.widgetBackgroundColor` | `--joy-bg-primary`, `--joy-surface` |
| `theme.blockBgColor` | `--joy-bg-secondary`, `--joy-bg-tertiary`, `--joy-block-bg-color` |
| `theme.textColor` | `--joy-text-primary`, and `--joy-text-secondary` / `--joy-text-tertiary` / `--joy-border-light` derived by `color-mix` against the widget background |
| `theme.blockTextColor` | `--joy-block-text-color` |
| `theme.cardHeaderColor` | `--joy-card-header-color`, `--joy-heading-text-color` |
| `theme.cardBorderColor` / `cardBorderWidth` | `--joy-card-border-color`, `--joy-card-border-width` |
| `theme.cardHoverColor` | `--joy-block-bg-hover` |
| `theme.secondaryButtonColor` / `TextColor` | `--joy-secondary-bg`, `--joy-secondary-text`, `--joy-secondary-hover` |
| `theme.footerBgColor` / `footerActiveBgColor` / `footerActiveTextColor` / `footerInactiveTextColor` | `--joy-footer-*` |
| `fonts.body` / `fonts.heading` | `--joy-font-family`, `--joy-font-family-heading` |
| `fonts.bodyWeight` / `headingWeight` | weight token pairs (normal+medium, semibold+bold) |

Notes that matter:

- Unset values fall through to `tokens.css` defaults via conditional spread. Do
  not invent a value the merchant has not set.
- Hover, secondary and muted colours are **derived**, not configured. Do not
  present them as separate choices.
- Button label colour auto-contrasts against the button fill (`isColorLight`).
- Font `'inherit'` means the storefront font wins, resolved on load.

**Anything not in this table is not merchant-configurable.** Layout, spacing,
component structure, border radii beyond `cornerRadius`, and the icon set are
fixed by the library.

## Running the real widget

Prefer this over any recreation. The package's own dev server compiles from
source:

```bash
cd /Users/avada/WebstormProjects/joy/packages/web-components
npx vite --port 5199 --strictPort
```

Put your demo `.html` inside that package so it can import the source entry, and
load it with `import '/src/index.ts'`. Then open
`http://localhost:5199/<your-file>.html`.

The prebuilt `dist/joy-loyalty.js` also exists but is a snapshot, may be stale,
and lazy-loads `./chunks/*` — copying the single file out of `dist/` gives you a
page where `customElements.get('joy-loyalty-widget')` is `false`. Copy
`dist/`, `dist/chunks/` and `dist/assets/` together, or just use the dev server.

Always verify in the browser that the element upgraded before reporting success:

```js
({ defined: !!customElements.get('joy-loyalty-widget'),
   shadow: !!document.querySelector('joy-loyalty-widget')?.shadowRoot })
```

If you cannot open a browser, say the demo is unverified. Do not claim it works.

## Widget usage, from `index.html`

Attributes are kebab-case; list data is set as **properties**, not attributes.

```html
<joy-loyalty-widget
  variant="drawer"           <!-- settings.display.type -->
  position="left"            <!-- settings.launcher.position -->
  program-layout="tabs"      <!-- settings.display.programLayout -->
  mobile-height="eighty-percent"
  corner-radius="radiusSoft" <!-- settings.theme.cornerRadius -->
  logged-in
  customer-name="Sarah" points="2480" points-label="Points"
  current-tier="Silver" next-tier="VIP" tier-progress="75"
  progress-type="spend"      <!-- match tierSettings.entryMethod -->
  exclusive-tier             <!-- shows an exclusive tier in the ladder -->
  show-referral show-tiers
  primary-color="..." widget-background-color="..." header-background-color="..."
  block-bg-color="..." card-header-color="..." >
</joy-loyalty-widget>
```

```js
await customElements.whenDefined('joy-loyalty-widget');
const w = document.querySelector('joy-loyalty-widget');
w.earnPrograms   = [...];  // {id, icon, title, event, description, detailDescription, value, points, actionLabel, actionUrl}
w.redeemPrograms = [...];  // {id, icon, title, event, description, points, value, discount, redeemType, stepPoints, earnAmount}
w.open();
```

`exclusive-tier` is a real property on the widget (`joy-loyalty-widget.ts`), which
is how an `isExclusiveTier` tier becomes visible.

Widget width is `--joy-widget-width: 380px` on desktop; under 768px it becomes a
full-width bottom sheet with `--joy-widget-border-radius: 0`, height driven by
`mobile-height`.

## Icons

Use the names in `src/icons/index.ts`. They are outlined stroke SVGs sized
12/16/20/24/32/48/64/80 by `joy-icon` size. Real names include:

`star star-filled gift ticket sparkle coupon tag percent dollar-sign
free-shipping credit-card coins cart shopping-bag trophy crown award medal
diamond shield user user-plus users share heart calendar clock repeat mail
home orders chat copy link globe instagram facebook twitter tiktok youtube`

When mirroring the widget outside the real components (a Figma or Framer comp),
match this vocabulary rather than inventing icons. Lucide is the closest
public set in style.

## The loyalty page is also components

`components/loyalty-page/` ships `joy-loyalty-page` and its bands. Before
authoring `joy-*.liquid` sections from scratch via `joy-loyalty-build`, check
whether these cover the design, and read
`docs/features/widget-v4/loyalty-page-migration.md`.

This matters most when `shop.canUseJoySDK` is false: these components do not go
through `window.joyInstance`, so they keep working where an SDK-driven custom
section would not.

## Plan gating

`shop.canUseJoySDK: false` means `window.joyInstance` is a **Proxy returning
stubs**, not a missing object: `customer()` resolves `null`, `earnPrograms()`
resolves `[]`, `redeem*` resolve `{success:false, message:'Ultimate plan
required'}`, and void methods like `openWidget` return `undefined`. Only
`triggerActivity` survives on Advanced (`shared/helpers/sdkAccess.js`).

So a section that "gracefully handles" a missing SDK still renders empty. Read
`window.AVADA_JOY` (server-rendered, already carries programs, tiers and
settings) or use the web components. Check this flag before choosing an
architecture, not after building.

## Opening the widget from your own section

**`window.avadaJoyTrigger()`.** Joy's floating button assigns it
(`packages/scripttag/src/components/Preview/FloatingButton.js`), and Joy's own
loyalty-page component calls exactly this
(`components/loyalty-page/joy-how-it-works.ts`). It is a plain global, so it
works regardless of plan gating.

Two things that do **not** work, and are easy to assume:

- **A `joy:open` event.** Nothing in the Joy source listens for one. Grep before
  dispatching any `joy:*` event; the ones that exist are emitted *by* Joy, not
  consumed by it.
- **`joyInstance.openWidget()`.** It internally calls `avadaJoyTrigger`, but the
  SDK proxy intercepts it first on a gated shop and returns `undefined`.

The trigger only exists once the widget script has booted, so a click handler
should poll briefly rather than fail on the first press, and fall back to a real
URL if it never appears. A button that silently does nothing is worse than one
that navigates somewhere sensible.

## Deep-linking a widget screen

`avadaJoyTrigger()` opens the widget on its home view. To land on a **specific**
screen, Joy routes by URL hash. The map is in `_handleAutoOpen`
(`packages/scripttag/v4-adapters/widget/WidgetAdapter.js`):

| Hash | Opens |
|---|---|
| `#joy-home` | main |
| `#joy-ways-to-earn` | main, earn tab |
| `#joy-ways-to-redeem` | main, redeem tab |
| `#joy-referral-program` | the referral screen |
| `#joy-open` | opens, no particular view |
| `#joy-loyalty` | opens, members only |

Two traps, both easy to hit:

- **The names are not the obvious ones.** It is `#joy-referral-program`, not
  `#joy-referral`. Read the map, do not guess from the view name.
- **`_handleAutoOpen` runs once at widget init, and nothing listens for
  `hashchange`.** Writing the hash on an already-loaded page therefore does
  nothing by itself. It only works on load or reload.

So an in-page control that should land on a screen does both: writes the hash
(so the URL is shareable and a reload behaves), and drives the element directly.

```js
var el = document.querySelector('joy-loyalty-widget');
history.replaceState(null, '', location.pathname + location.search + '#joy-referral-program');
el.open();
el.navigateTo('referral');   // view names, not hash names
```

For the earn and redeem screens, note that they are **tabs on the main view**,
not views of their own. The adapter navigates to `main` and then sets
`el._activeTab = 'earn' | 'redeem'`, so a control that deep-links there must do
the same rather than passing a view name that does not exist.

`navigateTo(view)` is public on `joy-loyalty-widget`. The view names are the
`_currentView` union in `joy-loyalty-widget.ts`: `main`, `referral`, `coupons`,
`activities`, `profile`, `orders`, `perks`, `wallet-pass`, and others.

Use `replaceState` rather than assigning `location.hash`: assignment makes the
browser jump to any element that happens to share the id, and stacks a history
entry so Back returns to the same page.

**Verify every event and method name against the source before wiring a control
to it.** An invented name produces a button that looks finished, passes review,
and does nothing. That is the most expensive kind of bug to ship.
