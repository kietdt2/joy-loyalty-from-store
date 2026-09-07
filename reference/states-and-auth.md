# States, auth, and what changes when a customer signs in

A loyalty page is read by four different people: a stranger, a member who just
arrived, a member the page cannot reach, and a member whose data is still in
flight. Most loyalty pages are built for the third of those and break for the
other three.

Audit this **before** you call a build finished. A band that renders correctly
signed-in and collapses signed-out has not been reviewed.

## Signed in is not the same as being a member

With opt-in enrollment on, a customer can be signed in to Shopify and still not
be in the loyalty programme. `joyInstance.customer()` returns them with
`type: 'guest'` rather than `'member'`.

The app's rule, from `packages/functions` `shopUseAssignMember.js`:

```js
isOptInEnrollment(shop) = shop.isOptInEnrollment === true
                          && shop.customerLoyaltyEligibility !== 'manually_assigned_only'
                          && plan is tier 1
isOptInGuest(shop, c)   = isOptInEnrollment(shop) && c.type !== 'member'
```

So there are **three** audiences, not two:

| Who | What they need |
|---|---|
| signed out | create an account |
| signed in, `type: 'guest'` | **join**, one tap, they already have the account |
| `type: 'member'` | their data |

Showing the middle group a member view gives them a zero balance and no way to
fix it, which reads as broken. Showing them the signed-out view asks them to
create an account they already have. Both are wrong, and both are the default if
you only check `isSignedIn()`.

Resolving this needs **two** reads, from two different places, and getting
either one wrong produces the bug:

**1. Does the shop use opt-in?** `AVADA_JOY.shop.isOptInEnrollment`, which is on
the storefront blob alongside `canUseJoySDK`, `plan` and
`customerLoyaltyEligibility`. When it is false there is nothing to join, so a
join prompt sends the customer to a widget that cannot enrol them. Gate on this
first and skip the lookup entirely.

**2. Has this customer joined?** `customer.type`, and **only**
`joyInstance.customer()` has it. The Liquid-rendered blob carries the Shopify
customer drop, which has no `type` field at all. There is no server-rendered
fallback for this one field.

```js
if (!isSignedIn())        return 'guest';
if (!optInEnrollment())   return 'member';        // nothing to join
return sdkCustomer().then(c =>
  !c || !c.type ? 'unknown' : (c.type === 'member' ? 'member' : 'not-member'));
```

**The SDK loads after `AVADA_JOY`.** A membership check that runs the moment the
metafield blob is ready finds no SDK and gets nothing back. Poll for
`window.joyInstance` rather than reading it once.

**Never render the member view server-side for a signed-in visitor.** Liquid
knows there is a session; it cannot know there is a membership. Render hidden,
resolve, then reveal. Revealing first and correcting flashes a zero balance at
exactly the customer this is meant to protect.

When the SDK never answers, pick the failure that is merely redundant rather than
the one that looks broken: show the member view and log a warning, rather than
nagging a real member to join. But do not let "unknown" masquerade as a
confident "member" in your code, or the bug becomes invisible.

**Joining needs the customer HMAC** (`POST /customer/join-program` via
`joinLoyaltyProgram`), so a theme section should not sign that call itself. Open
the widget on `#joy-open` and let it own enrollment.

## The five states

Every SDK-driven band owns five and shows exactly one:

| State | Means | Must show |
|---|---|---|
| `loading` | data in flight | a skeleton at roughly the final height |
| `guest` | no Shopify session | an invitation and a join path, never an empty shell |
| `ready` | data arrived | the real content |
| `empty` | signed in, merchant configured nothing | an honest message, not "0 undefined" |
| `error` | the read failed | readable text, never a blank band |

Add a sixth, `join`, on any band that shows member data when the shop uses
opt-in enrollment.

**`empty` and `error` are different answers and must not share a message.** Empty
means the merchant has not set this up; error means you could not read it.
Telling a customer "no tiers yet" after a failed request is a lie they will act
on.

**`loading` is the server-rendered default.** A band that never boots then stays
on its skeleton rather than flashing empty, which is the honest failure mode.
Drive the switch from one attribute so a band cannot show two states or none:

```css
.joy-oneup__state { display: none; }
.joy-oneup[data-joy-view="ready"] .joy-oneup__state--ready { display: block; }
/* one rule per state */
```

## Skeletons that do not jump

A skeleton exists to hold the layout, not to look busy. Rules:

- **Render it server-side, inside the real container**, as that container's
  actual children. Then replacing them is a swap, not an insertion, and the page
  does not reflow.
- **Mirror the real card's shell**: same padding, radius, border, gap. Only the
  content is grey blocks.
- **Match the real count** where you know it. If the band caps at six programs,
  render six skeleton cards.
- Animate `opacity` only, and collapse it under `prefers-reduced-motion`.

## Auth is a Shopify signal, twice

Detect the session **server-side in Liquid** so classic accounts never flash the
wrong state:

```liquid
{%- liquid
  assign signed_in = false
  if customer != blank and customer.id != blank
    assign signed_in = true
  endif
-%}
```

Then **again client-side**, because New Customer Accounts and cache-served pages
do not populate the Liquid `customer` object:

```js
window.ShopifyAnalytics?.meta?.page?.customerId || window.__st?.cid
```

Never gate auth on `AVADA_JOY.customer` or `joyInstance.customer()`. Those mean
Joy **membership**, not a Shopify session: a signed-in visitor can be unknown to
Joy, and gating on them hides the page from real customers.

Mark elements `data-joy-show-when-signed-in` / `data-joy-hide-when-signed-in`,
render the right one hidden server-side, and add
`.thing[hidden] { display: none !important; }` because a bare `hidden` attribute
loses to the element's own `display`.

## Not every band needs auth

Tiers, ways to earn and ways to redeem are for the person deciding whether to
join. Showing a guest an invitation instead of the ladder removes the reason to
sign up. Only gate the parts that are **about this customer**: their balance,
their tier, their completed programs.

## What should change on sign-in

This is the part most builds skip. Signing in should visibly change the page,
not just unlock a number. And with opt-in enrollment there are **three** columns,
not two:

| Band | Signed out | Signed in, not a member | Member |
|---|---|---|---|
| Opening | headline, join and sign-in | "you are signed in, join to start earning" + a join button | greeting by name, live balance |
| Member status | invitation to create an account | join invitation, one tap | balance, tier, progress |
| VIP tiers | the ladder | the ladder | the ladder with **their** tier highlighted |
| Ways to earn | every program | every program | completed ones marked done |
| Referral | the offer | the offer, no account line | the offer plus which account is credited |

**Every band that shows member data needs the middle column.** Getting it wrong
is not a cosmetic bug: an un-joined customer sees a zero balance, concludes the
programme is broken or that they earned nothing, and leaves. Apply the check
everywhere, not just on the one band you happened to think about first.

Anything that credits a loyalty account, like the referral email line, should
hide for the middle column rather than name an account that cannot receive it.

## Where each piece of data lives

Two sources, and the split matters for perceived speed:

**`window.AVADA_JOY`** is server-rendered, always present, no request:
`customer` (the Shopify drop: id, email, first_name), `points` (metafield
snapshot), `tier` (vipTier metafield), `program.earning`, `program.spending`,
`allTiers`, `settings`, `moneyFormat`.

**`window.joyInstance`** is the SDK, and carries what a metafield cannot:
`customer()` for the live balance and the earned-program flags,
`hasEarnProgram({customer, program})` for completion.

**Render from `AVADA_JOY` first, then enrich from the SDK.** No band should wait
on the network to show anything. Paint the programs immediately with nothing
marked done, then mark them when the customer resolves. The reverse, marking
everything undone and correcting, is visibly wrong for a moment.

Check `shop.canUseJoySDK` before relying on the SDK, and guard every call
regardless: on a gated shop it is a Proxy returning stubs, so an unguarded call
resolves to `null` forever rather than throwing something you would notice.
-> `reference/joy-app-source.md`

## Completed earn programs

`joyInstance.hasEarnProgram({customer, program})` is the app's own answer and a
pure function. It returns **true when the program is still available**, so
completed is its inverse. Read it rather than reimplementing the rules, which
differ per event type (birthday is per-year, Google review is per-status, social
shares are repeatable).

Mark completed programs **dimmed and badged, not hidden**. A customer should be
able to see what they have already done; removing it makes the list look shorter
every visit and hides the evidence that the program works.

## The sixth state nobody tests: the theme editor

A band that renders correctly on the storefront can be broken in the editor, and
the merchant meets the editor first.

When a setting changes, Shopify **replaces the section's DOM**. Every node your
JS rendered is destroyed, and the IIFE that would rebuild it already ran at page
load and will not run again. The merchant changes a heading, the tiers vanish,
and the only fix they find is reloading the editor. It reads as a broken
section, because it is one.

The event is `shopify:section:load`, with the section id at
`event.detail.sectionId`. Wrap it once, in the shared bridge, so no section has
to remember:

```js
var inits = {};

function register(sectionId, fn) {
  if (!sectionId || typeof fn !== 'function') return;
  inits[sectionId] = fn;
  ready(function () { fn(); });
}

if (window.Shopify && window.Shopify.designMode) {
  document.addEventListener('shopify:section:load', function (event) {
    var fn = inits[event.detail && event.detail.sectionId];
    if (fn) fn();
  });
}
```

Then every SDK-driven section uses `register('{{ section.id }}', run)` rather
than `ready(run)`.

Three things this exposes:

- **The init has to re-query its own DOM on every call.** A setup function that
  captured node references at load time will run again and write into detached
  elements, which is worse than not running: it fails silently.
- **A reveal-on-scroll observer needs the same treatment.** Rebuilt nodes come
  back at `opacity: 0.01` having never been observed, so the section is present
  and invisible. Reveal them outright on `shopify:section:load` rather than
  re-observing — they are already on screen.
- **Global once-guards become bugs.** `if (window.__thingDone) return;` is
  correct for a delegated document-level listener, which survives the rebuild,
  and wrong for anything that queries elements, which does not. Audit each one
  by asking whether it touches the DOM.

Guard the listener with `window.Shopify.designMode` so no storefront visitor
pays for it.

## The audit

Run this before calling a page finished, and put the result in the report:

```
band          guest  signed-in  loading  empty  error  editor
opening         .        .         -       -      -      .
member status   .        .         .       .      .      .
vip tiers       n/a      .         .       .      .      .
ways to earn    n/a      .         .       .      .      .
ways to redeem  n/a      n/a       .       .      .      .
redemption      .        .         .       .      .      .
referral        .        .         -       -      -      .
```

`editor` means: change a setting on that section in the theme editor and confirm
the band re-renders without a reload. Any band with JS-rendered content needs a
tick here.

`n/a` is a legitimate answer where a band is the same for everyone, but write it
down deliberately rather than leaving a blank you never considered.
