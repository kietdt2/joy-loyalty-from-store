# SDK wiring: which band gets which call

Every number on a loyalty page is live data. A Figma frame showing "2,450 points"
is a mock, and shipping it as static text is the single most common way this
build goes wrong.

Detail lives in `resources/joy-sdk/`. This file is the per-band routing table plus
the traps you must not trip. Cite source the way those docs do (`Joy.js:736`,
`apiHookV1Popup.js:55`) and never assert a method, key, endpoint or event that is
not in `resources/joy-sdk/` or verifiable in the bundled sample.

## 1. The boot contract, identical in every SDK-driven band

Exactly two Joy globals exist: `window.AVADA_JOY` (bootstrap config, set at
`app-data.liquid:5`) and `window.joyInstance` (`index.js:121`). The only ready
event is `joy:ready`, dispatched **on `window`** once, after
`joyInstance.initialize()` resolves, with `detail: {joyInstance}` (`index.js:130`).

> **`joy:ready` is dispatched on `window`.** `document.addEventListener('joy:ready', ...)`
> never fires: the event's target is `window`, and it does not propagate down to
> `document`. The bundled sample's `joy-cb-status.liquid` listens on `document`
> (line 324) and only works because its 250ms polling fallback catches the
> instance. The sample's other four SDK-driven sections listen on `window`. Listen
> on `window`.

All three of these, always. Subscribe, immediate check, timeout:

```js
function whenJoyReady(cb, { timeout = 10000 } = {}) {
  let done = false;
  const finish = (instance) => { if (done) return; done = true; cb(instance); };
  window.addEventListener('joy:ready', (e) => finish(e.detail?.joyInstance || window.joyInstance));
  if (window.joyInstance && typeof window.joyInstance === 'object') finish(window.joyInstance);
  setTimeout(() => { if (!done) finish(null); }, timeout);   // degrade, do not hang
}
```

Re-read `window.joyInstance` inside the handler rather than trusting the captured
`e.detail.joyInstance`: the plan-gating Proxy can replace the global after
`joy:ready` fired (`index.js:135-141`).

**Plan gating is silent.** On a non-SDK plan the Proxy stubs return
(`sdkAccess.js:37-53`): `customer()` resolves `null`, `earnPrograms()` resolves
`[]`, `redeemPrograms()` resolves `{success:false, message:'Ultimate plan required'}`
(an object, not an array), `redeem` resolves `{success:false, message:'Ultimate plan required'}`,
`hasEarnProgram` returns `false`, `generateLinkReferral` returns `null`. So
`typeof` guard every method before calling it, `Array.isArray` guard every list,
and never let a stub value render as an empty grid with no explanation.

**Errors resolve, they do not throw.** The transports do not check HTTP status:
`fetchRequest` (GET) and `makeRequest` (POST) resolve with the parsed JSON body
even on 4xx and 5xx, and reject only when the body fails `JSON.parse`. Inspect the
resolved value in this order:

1. `resp === null` (`ApiManager.getData` swallowed its own error, `ApiManager.js:209-212`).
2. `resp.error` string.
3. `resp.success === false`.
4. `{status: false, error}`, the `#_error` shape returned rather than thrown by
   most `Joy` methods on a client-side guard failure (`Joy.js:199-201`).
5. A real promise rejection, which only happens on the `#_tryToReturn` read
   getters, and rejects **with the bare `#_error` method reference**, not with an
   object (`Joy.js:203-206`). `.catch(e => e.error)` yields `undefined` there.
   `customer()`, `tiers()`, `earnPrograms()`, `redeemPrograms()`, `rewardList()`,
   `referredHistory()` can all reject this way.

Practical guard, the one the themes use:

```js
if (result && result.status === false && result.error) throw new Error(result.error);
```

Surface the real server message in the error state. A generic "Something went
wrong" hides `Ultimate plan required`, which is the message that actually tells
the merchant what to do.

## 2. Per-band routing

`joy` below is `window.joyInstance`.

| Band | Call | Returns | Endpoint | Citation |
|---|---|---|---|---|
| Member status | `joy.customer()` | `data.customer` | GET `/info` | `Joy.js:589` |
| Member status, tier progress | `joy.tiers()` | the full `data` body | GET `/tierRewards` | `Joy.js:679` |
| Member status, claimable list | `joy.redeemPrograms()` | `data.spending`, plan-filtered | GET `/programs` | `Joy.js:635` |
| Ways to earn | `joy.earnPrograms()` | `data.earning`, plan-filtered | GET `/programs` | `Joy.js:656` |
| Ways to earn, completion | `joy.hasEarnProgram({customer, program})` | boolean, synchronous, no network | none | `Joy.js:815-846` |
| Ways to earn, birthday | `joy.updateDOB(month, day)` | `{status:true}`, guest-gated | POST `/customer/birthday/{id}` | `Joy.js:746` |
| Ways to earn, per event | `joy.handleEarningProgram({program, customer})` | `{status:true}` or `{status:false, error}` or `{status:false, shouldOpenModal:true}` | POST `/customer/social` or `/customer/newsletters`, or none | `Joy.js:848-988` |
| Ways to redeem | `joy.redeemPrograms()` | `data.spending` | GET `/programs` | `Joy.js:635` |
| Ways to redeem, spend | `joy.redeem(id, points = 0, variantIdSelected)` | `{discount: {code, rawId, ...}, customer: {point, ...}}`; dispatches `joy:redeemCoupon`; guest-gated | POST `/redeem` | `Joy.js:736`, route `apiHookV1Popup.js:55` |
| Coupon list, revoke | `joy.revokeCoupon(rewardId)` | `{success, data}`; dispatches `joy:revokeCoupon` | POST `/refund-coupon` | `Joy.js:743`, route `apiHookV1Popup.js:48` |
| Coupon list, read | `joy.rewardList({customerId, limit, ...})` | the FULL body, no `.data` unwrap | GET `/rewards` | `Joy.js:626`, route `apiHookV1Popup.js:47` |
| VIP tiers | `joy.tiers()` | tiers with `tierRewards` attached | GET `/tierRewards` | `Joy.js:679` |
| Referral, config | `joy.referralProgram()` | `data.referralProgram` | GET `/info` | `Joy.js:605` |
| Referral, link | `joy.generateLinkReferral(email)` | `{urlReferral}`, never `url`, see section 6; dispatches `joy:referralGenerated`; guest-gated | POST `/referral/generateLinkReferral` | `Joy.js:749` |
| Referral, history | `joy.referredHistory({customerId, before, after, ...})` | cursor-paginated response | GET `/referred` | `Joy.js:671` |
| Referral, email a friend | `joy.sendReferralEmail({refereeEmail, emailCustomer})` | response | POST `/referral/sendEmail` | `Joy.js:752` |
| Anything else | `joy.get({path, cacheKey, customerId})` / `joy.post(path, data)` | pass-through | prefix `/app/api/v1/popup` | `Joy.js:991` / `Joy.js:990` |

Notes that bite:

- `get()` takes an **object** (`{path}`), `post()` takes **positionals**
  (`path, data`). `joy.get('/tiers')` is a dead branch that appears in the wild.
- `rewardList()` does not unwrap `.data`, and there is **no `coupons` key**. Read
  it as `const rewards = Array.isArray(resp?.data) ? resp.data : [];`
- `revokeCoupon` takes the **reward** id (`reward.id` / `reward._id` off
  `rewardList()`), not the program id. Passing a program id refunds nothing
  (`resources/joy-sdk/reference/redeem.md` section 6).
- `redeem` resolves the inner `data`, so the coupon is `result.discount.code` and
  the post-spend balance is `result.customer.point`. Themes read the code
  defensively as `result?.data?.discount?.code || result?.discount?.code`
  (`resources/joy-sdk/reference/redeem.md` section 3.3). `points` is used **only** when
  `program.redeemType === 'dynamic'`; a fixed program ignores it and spends
  `program.spendPoint`.
- `tiers()` returns the full body, so normalize:
  `Array.isArray(t) ? t : (t && t.data) || []`.
- **Tier progress is `customer.tierPoint`, never `customer.point`.** Two
  separate mistakes hide here and a member status band usually ships with both.

  **1. The wrong customer field.** `point` is the *spendable* balance: it falls
  the moment the customer redeems a reward. Driving tier progress from it
  demotes people for using the programme, which is the opposite of what a tier
  ladder is for. `tierPoint` is the tier-assessment value and is not spent
  down. If `customer.nextDeductedTierAmount` is present, subtract it, because it
  is a pending deduction against that balance; it is frequently absent, so guard
  rather than assume. Ref: `redeem.md` 411-413,
  `payloads/joy-customer.json`.

  **2. Assuming the axis is points at all.** `tierSettings.entryMethod` decides
  what the thresholds measure: `pointEarned`, `moneySpent`, `numberOfOrders`,
  `moneySpentB2B`, `moneySpentLifetime`. Resolve it as
  `AVADA_JOY.tierSettings ?? AVADA_JOY.program.tierSettings ?? {entryMethod:'pointEarned'}`.

  **`targetPoint` is the threshold in the units implied by `entryMethod`, in
  spite of its name.** On a `moneySpent` shop `targetPoint: 250` means 250
  *currency units*, not 250 points, and Joy accumulates the qualifying spend
  into `tierPoint` itself. So the comparison stays
  `tierPoint >= targetPoint` on every entry method: do **not** switch the value
  to `totalSpent` or `ordersCount`. Those are separate lifetime counters and
  they drift from the tier assessment as soon as a reset window or a demotion
  applies.

  What `entryMethod` changes is only the **wording and formatting**:

  | entryMethod | Remaining reads |
  |---|---|
  | `pointEarned` | `120 points to Insider` |
  | `moneySpent` | `$120 to Insider` |
  | `numberOfOrders` | `3 orders to Insider` |

  Apply the same unit to the threshold labels on the tier cards and the scale
  row. Shipping "From 250 points" on a `moneySpent` shop states a number the
  customer can never reconcile with their own account.

  This is a known cross-variant bug, not a theoretical one:
  `theme-patterns.md` records that v2 status cards compute the progress bar from
  the entry-method value while the remaining figure always reads `tierPoint`,
  so the bar and the sentence disagree on money and order shops.
- Hero, nav, how-it-works, FAQ and closing CTA make **no SDK call at all**. Their
  only dynamic behaviour is the auth gate, which is a Shopify signal.

## 2.0 Render the list from the SDK, never from hand-authored blocks

A ways-to-earn or ways-to-redeem band whose rows are theme blocks is wrong the
day the merchant adds a programme. The list must come from `earnPrograms()` /
`redeemPrograms()` so a new programme appears on its own, with no theme edit.

Blocks still earn their place, but only as the **fallback and first paint**: the
band says something true before the network answers, and it still says it with
JavaScript disabled. Once the SDK responds, replace the whole list. Keep the
authored rows when the response is empty rather than blanking the band.

```liquid
<ul data-joy-list>
  {%- for block in section.blocks -%}...{%- endfor -%}   {# fallback only #}
</ul>
```

**The raw payload is not the list to render.** Neither
`AVADA_JOY.program.earning` nor `earnPrograms()` applies the merchant's "Show on
loyalty page" toggle, so an unfiltered render shows more rows than the Joy
widget does. Replicate `prepareEarningProgram`:

```js
programs
  .filter(p => p.status === true && p.isDraft !== true)
  .filter(p => p.showLoyaltyPage === true || p.showLoyaltyPage === 'true')
  .filter(p => !(p.event === 'milestone' && p.typeMilestone === 'milestone_inactivity'))
  .filter(p => p.expired !== true)
  .sort((a, b) => (a.displayOrder ?? a.priority ?? 999) - (b.displayOrder ?? b.priority ?? 999));
```

Write the figure the way the customer reads it, branching on `earnBy`:

| Program | Fields | Rendered |
|---|---|---|
| `sign_up` | `earnPoint: 100` | `100 points` |
| `place_order` | `earnPoint: 4`, `rateMoney: 1`, `earnBy: 'price'` | `4 points per $1` |

A bare `100` or `4x` makes the reader supply the unit. Spell out "points", and
take the currency symbol from `shop.money_format` rather than hardcoding `$`.

**Let the row count drive the layout, but keep the tracks equal.** A grid fixed
at three columns leaves a visible hole when only two programmes exist, and the
pair reads as left aligned under a centred heading. The fix is *not*
`grid-auto-flow: column` with `grid-auto-columns`: that sizes each track to its
own content, so a row with a short label and a row with a long one come out
visibly different widths and the figures stop lining up.

Use a `1fr` repeat matched to the count, capped at three per row, and centre the
grid with its own `max-width` and `margin-inline: auto`:

```css
.grid { grid-template-columns: repeat(3, minmax(0, 1fr)); max-width: 1000px; margin-inline: auto; }
.grid:has(> :last-child:nth-child(1)) { grid-template-columns: minmax(0, 340px); max-width: 340px; }
.grid:has(> :last-child:nth-child(2)) { grid-template-columns: repeat(2, minmax(0, 1fr)); max-width: 720px; }
```

`:last-child:nth-child(n)` is how you branch on "exactly n children" in CSS.
Three is the ceiling; a fourth programme wraps to a second row rather than
squeezing the tracks. `minmax(0, 1fr)` rather than bare `1fr` so a long unbroken
word cannot blow the track out.

**Match the surrounding bands rather than styling the rows in isolation.** These
action rows are the same shape as a how-it-works step: figure, rule, label,
detail. Reuse that band's type sizes so the page reads as one system instead of
two similar layouts at slightly different scales.

## 2.1 Band CTAs: deep link the widget with a URL hash

An earn or redeem band that only describes the programme leaves the customer
with nowhere to go. Give each one a link that opens the Joy widget already on
the matching screen, so the band explains and the widget transacts.

**Use the URL hash. It is a plain anchor and no JavaScript at all.** The widget
installs its own `hashchange` listener and routes on the literal hash
(`FloatingButton.js:174-230`), calling `openPage()` internally. Source of truth
is `packages/scripttag/src/const/deeplinkHash.js`:

| Hash | Widget screen | Page key |
|---|---|---|
| `#joy-loyalty` | widget home | `home` |
| `#joy-home` | widget home | `home` |
| `#joy-ways-to-earn` | ways to earn | `howToEarn` |
| `#joy-ways-to-redeem` | ways to redeem | `howToRedeem` |
| `#joy-referral-program` | referral step 1 | `referralStep1` |
| `#joy-activities` | points history | `activities` |
| `#joy-rewards` | my coupons | `rewards` |

```liquid
<a class="band__cta" href="#joy-ways-to-earn">Start earning</a>
```

That is the whole integration. Prefer it over scripting `openPage()` for three
reasons: it survives the widget mounting late, because the listener belongs to
the widget rather than to your section; it is shareable and bookmarkable, so
marketing can link straight to a screen from an email; and it needs no guard for
the mount-time race described below.

**Why not call `openPage()` directly.** It is a *mount-time* member: attached by
`FloatingButton.js` only after the widget's React tree mounts, so it is
`undefined` on `window.joyInstance` immediately after `joy:ready`, and a click
handler bound early throws. If you genuinely need the programmatic call, for
example `openRedeemProgram(id)` to open one specific reward on `redeemStep1`,
poll for it and fall back to `openWidget()`, the only opener `Joy.js` itself
guarantees:

```js
var openJoyPage = function (pageKey, attempt) {
  attempt = attempt || 0;
  var joy = window.joyInstance;
  if (joy && typeof joy.openPage === 'function') { joy.openPage(pageKey); return; }
  if (joy && typeof joy.openWidget === 'function' && attempt > 8) { joy.openWidget(); return; }
  if (attempt > 20) return;                    /* give up rather than spin */
  setTimeout(function () { openJoyPage(pageKey, attempt + 1); }, 150);
};
```

Two behaviours worth knowing. The hash effect is gated on `floatButtonVisible`,
so it fires once the widget is actually on the page rather than on load. And
`FloatingButton.js:754-758` skips resetting the widget to its initial page when
a hash is present, which is what stops the deep link being overwritten by the
default screen.

## 2.2 Auth-aware CTAs, and returning the customer to the page

Every sign-in and join CTA on a loyalty page carries a **return URL**. Without
one Shopify drops the customer on the account dashboard after login, losing the
page they were reading and the action they were about to take:

```liquid
href="{{ routes.account_login_url }}?return_url={{ request.path | url_encode }}"
```

Use `request.path` rather than a hardcoded handle so the link survives a page
rename. This applies to `account_login_url` and `account_register_url` alike.

A band whose CTA is only useful to one audience ships **both** buttons and hides
the wrong one, rather than rendering a single button whose label changes:

| Signed out | Signed in |
|---|---|
| Join now, to the login page | View more, to a collection |
| Sign in to get your link | the link itself |

Render both, hide the wrong one server-side with `{% if customer.id != blank %}`
so classic accounts never flash, and hide it again client-side for New Customer
Accounts. Auth stays a Shopify signal.

## 3. Bootstrap fallbacks, and why you need them

`earnPrograms()` and `redeemPrograms()` are plan-filtered. `window.AVADA_JOY.program`
is not. The classic symptom is a grid that is full for a guest and empty once the
member signs in. Both grids in the sample therefore merge the two sources.

| Key path | Use |
|---|---|
| `AVADA_JOY.program.earning` | ungated earning programs |
| `AVADA_JOY.program.spending` | ungated spending programs |
| `AVADA_JOY.program.tiers` | ungated tier list |
| `AVADA_JOY.tierSettings`, alias `AVADA_JOY.tierProgram` | tier program settings |
| `AVADA_JOY.program.tierSettings` | second fallback for the above |
| `AVADA_JOY.tierProgram.isNewSetupContentPerk` | the perk-generation switch, trap 3 |
| `AVADA_JOY.customer.id`, `.email` | the only customer fields the SDK itself reads |
| `AVADA_JOY.points` | a server-rendered metafield snapshot, may be stale or empty |
| `AVADA_JOY.tier`, `AVADA_JOY.tier.tierId` | from `customer.metafields.avada_joy.vipTier` |
| `AVADA_JOY.settings.pointSingular` / `.pointPlural` | point terminology, defaults `point` / `points` |
| `AVADA_JOY.shop.currency`, `AVADA_JOY.locale` | formatting |
| `AVADA_JOY.login_url` | guest redirect, default `/account/login` |
| `AVADA_JOY.status` | falsy means the SDK is disabled entirely (`index.js:98`) |

The Liquid-rendered `AVADA_JOY.customer` is
`{id, email, first_name, last_name, default_address, hash, isCustomerB2B, tags, phone, state, tax_exempt}`
and carries **no `point`** (`resources/joy-sdk/reference/runtime.md` section 1.1). Read the balance from
`customer()`, whose captured payload does carry `point`.

Entry method for tier progress:
`AVADA_JOY.tierSettings ?? AVADA_JOY.program.tierSettings ?? {entryMethod: 'pointEarned'}`.
Values are `pointEarned | moneySpent | numberOfOrders | moneySpentB2B | moneySpentLifetime`
(`resources/joy-sdk/reference/theme-patterns.md` section 4).

The measure must match the unit `targetPoint` is expressed in. **Three branches,
not two:**

| `entryMethod` | Measure | Format |
|---|---|---|
| `moneySpent`, `moneySpentB2B`, `moneySpentLifetime` | `customer.totalSpent` | currency |
| `numberOfOrders` | `customer.orderCount` | count plus an orders unit |
| `pointEarned`, and anything unrecognised | `customer.tierPoint ?? customer.point` | points |

Do not fold the orders branch into the points branch. `numberOfOrders` is the live
value in the captured payload (`joy-programs.json` `data.tierSettings.entryMethod`),
and a points-over-order-count bar computes 9800 / 3 instead of 3 / 5.
`templates/vip-tiers-section.liquid` and the sample's `joy-cb-tiers.liquid` branch
all three ways; the sample's `joy-cb-status.liquid` does not, and is wrong there.

## 4. The four states, per band

Build all four for every SDK-driven band. The design draws only the third one.
Ask the user for the states the design omits rather than inventing a look.

| State | Trigger | What it must be |
|---|---|---|
| Loading | before the first call resolves | a skeleton in the shape of the final content, shipped as the custom element's initial light-DOM children so it exists before any JS runs. Not a spinner, not the word "Loading" |
| Logged out | no Shopify session signal, or `customer()` returns null or has no `id` | a composed prompt plus one sign-in CTA. Not an error. Split it: signed in to Shopify but unknown to Joy gets the panel without a Join button |
| Empty | the call succeeded and the list is empty | says how to populate it ("Earn your first points by ..."). Never a blank box. For program grids this usually means the merchant has not configured programs yet, so say that |
| Error | any of the five failure shapes in section 1 | inline, with the **real** server message, plus a retry affordance where retrying can help |

Liquid-static bands have two states instead: signed out and signed in, both
server-rendered, both corrected client-side for New Customer Accounts.

**Auth is a Shopify signal, never a Joy signal.** Server:
`customer != blank and customer.id != blank`. Client, covering New Customer
Accounts and cache-served pages: `window.ShopifyAnalytics.meta.page.customerId`
or `window.__st.cid`. Do not gate auth on `AVADA_JOY.customer` or
`joyInstance.customer()`: those mean Joy loyalty membership, and a visitor can be
signed in to Shopify but unknown to Joy. Use Joy only for loyalty data.

Because of New Customer Accounts, SDK-driven bands should call `customer()`
**unconditionally**, even when the Liquid `customer` drop is empty.

## 5. The three traps

These are reproduced from `resources/joy-sdk/SDK-INDEX.md` because a builder who
trips them ships a bug that looks fine in review and is wrong in production.

### Trap 1: completion is the `earn<Event>` booleans, and `hasEarnProgram` is inverted

Priority order (`resources/joy-sdk/reference/earn.md` section 2):

```js
// 1. PRIMARY: SDK boolean. false = COMPLETED (cannot earn). true = can still earn.
const canEarn = joyInstance.hasEarnProgram({ customer, program }); // Joy.js:815-846, SYNC
const completed = canEarn === false;
// 2. FALLBACK A: per-event boolean on customer() -> earn<Event>
// 3. FALLBACK B: program-level isEarned / isCompleted, when present
```

**There is no `completedPrograms` array, no `signupComplete` field, and no
per-program `isCompleted` on `/programs`.** Any code reading those is dead. The
captured payload `resources/joy-sdk/reference/payloads/joy-customer.json` is the
authority on field names: 36 flat keys including `point`, `tierId`, `tierPoint`,
`hasTier`, `totalSpent`, `orderCount`, `referralCode`, `referralPendingCount`,
`earnSignUp`, `earnFollowInstagram`, `type`, `id`.

The flag names are a hand-written map (`getEarnFlag.js`), not a transform. Exactly
**two** of them break a naive `'earn' + PascalCase(event)`
(`resources/joy-sdk/reference/events-and-programs.md` section 1.8):

- `birthday` is **`earnBirthDayReward`**, not `earnBirthday`, and it counts as
  complete only when the current year equals `customer.lastYearGetBirthdayReward`.
- `join_whatsapp` is **`earnWhatsapp`**, not `earnJoinWhatsapp`.

Every other entry in that table does match the naive transform,
`sign_up_newsletter` to `earnSignUpNewsletter` included. Build your map from the
table in section 1.8, not from memory, and check it against the sample's
`_fallbackDone` in `joy-cb-earn.liquid`.

Some events are deliberately re-earnable and must never latch as completed
(`EVENT_SHARE_SOCIAL`, `events.js:48-55`): `share_twitter`, `share_facebook`,
`comment_instagram`, `story_mention_instagram`, `story_reply_instagram`,
`live_comment_instagram`.

And on a non-SDK plan the Proxy makes `hasEarnProgram` return `false`
unconditionally, which inverts to "everything is completed". Gate on
`typeof joy.hasEarnProgram === 'function'` **and** a real customer before
trusting it.

### Trap 2: `earnPointsTiers` is gated on the PROGRAM flag, not on the customer's tier

`earnPointsTiers[customer.tierId]` is only valid when `isAppliedVipTier` is true.
`place_order` V1 programs gate on `appliedPlaceOrderTo === 'vip-tier'` instead
(V2, meaning `isEventPlaceOrderV2` truthy, uses `isAppliedVipTier`). Programs keep
a **stale** `earnPointsTiers` map after the merchant switches the VIP-tier reward
off, so an ungated read shows correct base points logged out and inflated tier
points once signed in. That exact bug shipped as "15 pts logged out, 100 pts
logged in" on a write_review program.

Safe form, verbatim from `resources/joy-sdk/reference/earn.md` section 3:

```js
function tierOverride(program, customer) {
  if (!customer || !customer.tierId || !program.earnPointsTiers) return null;
  const ev = String(program.event || '');
  const vipFlag = program.isAppliedVipTier === true || program.isAppliedVipTier === 'true';
  const vip = (ev === 'place_order' || ev === 'place_order_subscription')
    ? (program.isEventPlaceOrderV2 ? vipFlag : program.appliedPlaceOrderTo === 'vip-tier')
    : vipFlag;
  return (vip && program.earnPointsTiers[customer.tierId]) || null;
}
function effectivePoints(program, customer) {
  const tc = tierOverride(program, customer);
  if (tc) {
    if (tc.isDisableReward) return 0;                    // this tier earns nothing
    if (tc.earnPoint != null) return Number(tc.earnPoint) || 0;
  }
  return Number(program.earnPoint ?? program.points) || 0; // base fallback
}
```

Shape is `{ [tierId]: { earnPoint, rateMoney?, rateItem?, extraPoints?, limitUnit?, isDisableReward? } }`,
and the values are sometimes strings (`"5"`), so coerce with `Number(...)`.

### Trap 3: VIP perks exist in two generations on the same tier object

Every tier from `/tierRewards` carries **both** the legacy `contentPerk[]` array
of strings and the new structured `tierRewards[]` objects. The new admin "Display
configuration" writes only `tierRewards`; `contentPerk` stays frozen at its
pre-migration value. A section that reads only `contentPerk` renders the old perk
list forever. That caused a live bug on two stores in 2026-07.

The app's own rule (scripttag `ContentPerk.js`):

> Render legacy only when some tier still ships `contentPerk` **and**
> `AVADA_JOY.tierProgram.isNewSetupContentPerk` is falsy. Otherwise render
> `tierRewards`, filtering `status !== false` and `showLoyaltyPage !== false`.
> Per tier, fall back to whichever model actually has data.

`perkVisibility === 'hidden_from_lower'` means the perk shows only at or above
the customer's own tier.

The captured payload proves the two disagree on the same tier: in
`resources/joy-sdk/reference/payloads/joy-tier-rewards.json` the Gold tier's
`contentPerk` has 7 entries including `"Bonus points"`, while its
`tierRewards[].title` has 6 and no `"Bonus points"`.

Canonical tier sort, used everywhere:

```js
tiersData.filter(t => !t.inactive).sort((a, b) => {
  const aPos = a.position !== undefined ? a.position : 999;
  const bPos = b.position !== undefined ? b.position : 999;
  return aPos !== bPos ? aPos - bPos : (a.targetPoint || 0) - (b.targetPoint || 0);
});
```

Current-tier resolution, four strategies that coexist in the wild. Chain them and
fail closed when a signed-in member's tier cannot be resolved:
`customer().tierId` matched to `tier.id` -> `AVADA_JOY.tier.tierId` -> the
`avada_joy.vipTier` metafield's `tierName` matched to `tier.name` ->
`targetPoint <= measure` threshold.

## 6. Referral, the fourth trap in practice

`generateLinkReferral(email)` returns the link as **`urlReferral`**, never `url`.
Alias it:

```js
const gen = await joy.generateLinkReferral(email);   // resolves the inner `data`
const link = gen
  ? (gen.url || gen.urlReferral || (gen.data && (gen.data.url || gen.data.urlReferral)) || '')
  : '';
```

The route body is `{data: {urlReferral}}` (`resources/joy-sdk/reference/endpoints.md`), and the SDK resolves the
inner `data`, so `gen.urlReferral` is the normal hit. The extra `gen.data.*` arm is
what `joy-cb-referral.liquid:449` ships, and it costs nothing.

A shipped variant omitted this alias and reported "No referral URL generated" on
every successful call. The sample's `joy-cb-referral.liquid` aliases correctly.

Only `customerRewardEarnAmount` is an attested reward field on
`referralProgram()`. The friend-side amount and the discount-type names are not
attested, so degrade to a generic label rather than printing a value the payload
may not carry.

Completed referrals come from `referredHistory({customerId, limit})`, counting
rows whose `statusReferral === 'complete_referral'`.

## 7. Merchant-side setup to report

Wiring the SDK correctly is not enough for the page to show anything. Say plainly
in the final report which of these the merchant must do:

- Avada Joy installed, enabled (`AVADA_JOY.status` truthy) and emitting `joy:ready`.
- Earning programs created and set to show on the loyalty page. Neither
  `earnPrograms()` nor the bootstrap is pre-filtered by the merchant's "Show on
  loyalty page" toggle, so the section applies that filter itself
  (`reference/ways-to-earn-modals.md` section 6).
- Spending programs created, with point costs, for the redeem grid.
- VIP tiers configured, with the entry method the progress bar assumes.
- The referral program enabled.
- Plan level: `redeem`, `generateLinkReferral` and friends are Ultimate-plan
  gated, and degrade to a message rather than an error.
- Per earn type, the integration it needs: Instagram for comment and story
  events, a Google review location, auto-approve settings.

## 8. Deeper reading

| Question | Doc |
|---|---|
| Method signature, return, error shape | `resources/joy-sdk/reference/sdk-api.md` |
| Globals, boot, web-component lifecycle, New Customer Accounts | `resources/joy-sdk/reference/runtime.md` |
| Earn fetch, completion, tier points, per-event triggers | `resources/joy-sdk/reference/earn.md` |
| Redeem flow, discount apply, revoke, rewards surfaces | `resources/joy-sdk/reference/redeem.md` |
| Event / program / tier / redeem-type literals | `resources/joy-sdk/reference/events-and-programs.md` |
| HTTP routes, HMAC, failure shapes | `resources/joy-sdk/reference/endpoints.md` |
| Which existing variant to copy, per section | `resources/joy-sdk/reference/theme-patterns.md` |
| Real field names in a response | `resources/joy-sdk/reference/payloads/` |
| A clean worked example per surface | `resources/joy-sdk/section-demo/` (11 sections) |
| The 13 earn-modal types, fixed logic vs themeable UI | `reference/ways-to-earn-modals.md` |
| The redeem modal contract, taxonomy, value math | `reference/ways-to-redeem-modals.md` |

Optional deeper ground truth, if the teammate happens to have them: the full Joy
app source at `/Users/avada/Documents/joy-app` and the theme collection at
`/Users/avada/Documents/joy/`. Neither is required. Everything above is verifiable
from what is bundled in `resources/`.
