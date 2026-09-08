# SDK API: `window.joyInstance`

Complete method reference for the storefront SDK instance (`window.joyInstance`, class `Joy`). Source of truth: `/Users/avada/Documents/joy-app/packages/scripttag/src/Joy.js` and its delegate `/Users/avada/Documents/joy-app/packages/scripttag/src/managers/ApiManager.js`. Signatures, params, return shapes, and endpoints below are read from source; ground-truth payloads live in [payloads/joy-customer.json](payloads/joy-customer.json) and [payloads/joy-programs.json](payloads/joy-programs.json).

`window.joyInstance` is a single `Joy` instance assigned synchronously, then `initialize()` runs, then `joy:ready` fires (`event.detail.joyInstance`). See [runtime.md](runtime.md) for boot order, globals, and plan gating. See [endpoints.md](endpoints.md) for the server-side endpoint table, [events-and-programs.md](events-and-programs.md) for dispatched events and program shapes, [earn.md](earn.md) for earning flows, [redeem.md](redeem.md) for redemption flows.

> WARNING: `#_`-prefixed members are ES private fields. They are NOT callable from theme code. Only the class-field arrow functions / methods listed here are the public surface.

> WARNING: Plan gating. On non-SDK plans (`isSdkAllowed(shop) === false`) `window.joyInstance` is a Proxy that returns safe stubs instead of running the real method (`redeem`/`handle`/`apply`/`update` -> `Promise.resolve({success:false, message:'Ultimate plan required'})`, `customer` -> `Promise.resolve(null)`, `earnPrograms` -> `Promise.resolve([])`, `redeemPrograms` -> `Promise.resolve({success:false, message:'Ultimate plan required'})` (NOT `[]`: `getSafeReturnValue` tests `.includes('redeem')` before `.includes('Programs')`, so `'redeemPrograms'` matches the redeem branch first), `hasEarnProgram` -> `false`, `generateLinkReferral` -> `null`, `sendReferralEmail` -> `{success:false, error:'Ultimate plan required'}`, else `undefined`). Advanced+ keeps only `triggerActivity`. Cite: `sdkAccess.js:37-53` (the `.includes('redeem')` branch at 38-45 precedes the `.includes('Programs')` branch at 50). Details in [runtime.md](runtime.md).

---

## 1. Master table

Host is `https://${process.env.API_URL}`. Default path prefix is `/app/api/v1/popup`; methods marked `widgets` use `/app/api/v1/widgets`. "via `#_tryToReturn`" means the method resolves the inner result if truthy, else **rejects** with the `#_error` method reference itself (a function, `typeof === 'function'`), NOT an object - the rejection value is the bare uncalled `#_error` method, so `.catch(e => e.error)` yields `undefined` (you get a function, not `{status:false,error:'No data found'}`). The `{status:false, error:'No data found'}` object is only produced when `#_error` is *invoked* for client-side guard failures that are RETURNED, not for `#_tryToReturn` rejections. Cite: `Joy.js:203-206` - `return new Promise((ok, err) => (result ? ok(result) : err(this.#_error)));` with `#_error` defined as a method at `Joy.js:199-201`.

| Method | Params | Returns | Endpoint | Notes |
|---|---|---|---|---|
| `initialize()` | none | `Promise<void>` | several | Boots widget; resolves before `joy:ready`. `Joy.js:51` |
| `updateCustomerState()` | none | `Promise<void>` | GET `/info` | Re-fetches `/info`, calls `window.avadaSetCustomerLogin(data.customer)`. Also `window.avadaInitAfterLogin`. `Joy.js:193` |
| `shop()` | none | `Promise<shop>` | GET `/info` | `data.shop`. `Joy.js:586` |
| `customer(url='/info', checkUseMetaField=false)` | `url`, `checkUseMetaField` | `Promise<customer>` | GET `url` | `data.customer`. Rejects if falsey. `checkUseMetaField=true` appends an `m` field-selector of already-loaded config keys (see detail below). See [payloads/joy-customer.json](payloads/joy-customer.json). `Joy.js:589` |
| `settingsBranding()` | none | `Promise<settings>` | GET `/info` | `data.settings`. `Joy.js:596` |
| `settingsReferral()` | none | `Promise<object>` | GET `/info` | `data.settingsReferral`. `Joy.js:599` |
| `referralProgram()` | none | `Promise<object>` | GET `/info` | `data.referralProgram`. `Joy.js:605` |
| `settingsLoyaltyQr()` | none | `Promise<object>` | GET `/info` | `data.settingWidgetQr`. `Joy.js:611` |
| `translation()` | none | `Promise<object>` | GET `/info` | `data.translation`. `Joy.js:617` |
| `tiers()` | none | `Promise<object>` | GET `/tierRewards` | Full `data` body (no field pick). `Joy.js:679` |
| `countMembersByTier(cacheKey='')` | `cacheKey` | `Promise<{tierId:count}>` | GET `/count-member-tier` | NOT via `#_tryToReturn`. `Joy.js:690` |
| `earnPrograms()` | none | `Promise<array>` | GET `/programs` | `data.earning`, pro-filtered. `Joy.js:656` |
| `redeemPrograms({query}={})` | `{query}` | `Promise<array>` | GET `/programs[?q]` | `data.spending`, pro-filtered. `Joy.js:635` |
| `rewardList({customerId,before,after,limit,...query}={})` | object | `Promise<resp>` | GET `/rewards?...` | Resolves customer if `customerId` omitted. `Joy.js:626` |
| `customerHistory({customerId,before,after,...query}={})` | object | `Promise<resp>` | GET `/activities?...` | Cursor pagination. `Joy.js:663` |
| `referredHistory({customerId,before,after,...query}={})` | object | `Promise<resp>` | GET `/referred?...` | `Joy.js:671` |
| `getCustomerMilestone({customerId}={})` | `{customerId}` | `Promise<data>` | GET `/customer-milestone` | `Joy.js:992` |
| `getInfoLoyaltyPage({customerId,hasMyRewardBlock})` | object | `Promise<object>` | GET `/loyalty-page?m=...` | NOT via `#_tryToReturn`. See pitfall below. `Joy.js:761` |
| `redeem(id, points=0, variantIdSelected)` | positional | `Promise<data>` | POST `/redeem` | Dispatches `joy:redeemCoupon`. Guest-gated. `Joy.js:736` |
| `revokeCoupon(rewardId)` | `rewardId` | `Promise<{success,data}>` | POST `/refund-coupon` | Dispatches `joy:revokeCoupon`. `Joy.js:743` |
| `updateDOB(month, day)` | positional | `Promise<{status:true}>` | POST `/customer/birthday/{id}` | Guest-gated. `Joy.js:746` |
| `generateLinkReferral(email)` | `email` | `Promise<data>` | POST `/referral/generateLinkReferral` | Dispatches `joy:referralGenerated`. Guest-gated. `Joy.js:749` |
| `sendReferralEmail({refereeEmail,emailCustomer}={})` | object | `Promise<data>` | POST `/referral/sendEmail` | `Joy.js:752` |
| `getCouponReferral({emailCustomer,emailFriend}={})` | object | `Promise<{discountCode}>` | POST `/referral/redeemCoupon` | Uses browser fingerprint. `Joy.js:755` |
| `triggerActivity(actionKey)` | `actionKey` | `Promise<resp>` | POST `/trigger-activity` | Guest-gated. Advanced+ allowed. `Joy.js:1001` |
| `handleEarningProgram({program, customer})` | object | `Promise<object>` | POST `/customer/social` or `/customer/newsletters`, or none | Multi-branch. `Joy.js:848` |
| `hasEarnProgram({customer, program})` | object | `boolean` | none (pure) | `Joy.js:815` |
| `calculateProductPoints({productId,variantId,quantity=1,customerId,locale=''})` | object | `Promise<{html,points,pointCurrencyName}>` | GET `/points-calculator/product?...` `widgets` | `Joy.js:1049` |
| `calculateCartPoints({items,autoFetchCart=false,customerId=null,locale=''}={})` | object | `Promise<{html,points,pointCurrencyName}>` | POST `/points-calculator/cart?...` `widgets` | Can auto-fetch `/cart.js`. `Joy.js:1094` |
| `get({path, cacheKey, customerId})` | object | `Promise<resp\|null>` | arbitrary GET | Pass-through to `apiManager.getData`. `Joy.js:991` |
| `post(path, data)` | positional | `Promise<resp\|{error}>` | arbitrary POST | Pass-through to `apiManager.postData`. `Joy.js:990` |
| `checkEarnCurrentPage(customer, programCustomTrigger)` | positional | `Promise<resp\|undefined>` | POST `/customer/social?...` | Visit-page earn; used by `initialize`. `Joy.js:1156` |
| `getDetectLocale(shop, translation)` | positional | `Promise<string>` | none | Detected non-primary locale or `''`. `Joy.js:807` |
| `registerRedirectGuest(callback)` | `callback(type, defaultUrl)` | `void` | none | Stores `guestRedirectCallback` in `window.joyRegistry`. `Joy.js:721` |
| `openWidget()` | none | `void` | none | `window.avadaJoyTrigger()` if closed. `Joy.js:620` |
| `closeWidget()` | none | `void` | none | `window.avadaJoyTrigger()` if open. `Joy.js:623` |
| `handleTTRCustomTiers(data)` | `data` | `void` | none | No-op in effect; init call site commented out. `Joy.js:147` |
| `getRegistryStats()` | none | stats or `null` | none | Request-registry cache stats. `Joy.js:1174` |
| `resetRegistryStats()` | none | `void` | none | `Joy.js:1181` |
| `clearRegistryCache()` | none | count | none | `Joy.js:1189` |
| `addScriptFreeProductToStoreFront(linkScript?)` | `linkScript` | `void` | loads script | Internal cart wiring. `Joy.js:181` (default `linkScript` param at 182) |
| `handleInsertFreeProductCart()` | none | `void` | various | Internal cart wiring. `Joy.js:158` |
| `addFreeProductChangeCart()` | none | `void` | patches `fetch` | Internal cart wiring. `Joy.js:167` |

Mount-time members (attached by `FloatingButton.js` only after the widget mounts, NOT in `Joy.js`): `openPage(page)`, `openRedeemProgram(id)`, `getPrograms()`, `setPrograms(next)`, `customerHistoryById({before,after})`, `rewardListById({before,after})`. See [runtime.md](runtime.md).

`openPage(page)` (`FloatingButton.js:801-804`) calls `openWidget()` then `setTimeout(() => nextPage(page), 100)`. The `page` arg is an internal widget page-key string from `const/widgetPage.js:1-4`:

| Constant | Page-key value | Deep-link hash | Source |
| --- | --- | --- | --- |
| `HOME` | `home` | `#joy-home`, `#joy-loyalty` | `widgetPage.js:1` |
| `HOW_TO_EARN` | `howToEarn` | `#joy-ways-to-earn` | `widgetPage.js:2` |
| `HOW_TO_REDEEM` | `howToRedeem` | `#joy-ways-to-redeem` | `widgetPage.js:3` |
| `REFERRAL_STEP_1` | `referralStep1` | `#joy-referral-program` | `widgetPage.js:4` |
| `MY_HISTORY` | `activities` | `#joy-activities` | `widgetPage.js:5` |
| `MY_COUPONS` | `rewards` | `#joy-rewards` | `widgetPage.js:6` |

**Prefer the hash over calling `openPage()` from theme code.** The widget
registers its own `hashchange` listener and maps each hash to its page key
(`FloatingButton.js:174-230`), so `<a href="#joy-ways-to-earn">` needs no script
and cannot lose the mount-time race below. Hash literals are defined in
`const/deeplinkHash.js`. `FloatingButton.js:754-758` deliberately skips the
reset-to-initial-page effect while a hash is present, so the deep link is not
overwritten by the default screen.

`openRedeemProgram(id)` (`FloatingButton.js:806-818`) calls `openWidget()`, then resolves the spending program by `id` (from `programs.spending`, lazily via `handleGetPrograms()` if not yet loaded), calls `setToRedeem(found)`, and navigates with `nextPage('redeemStep1')` - the page-key `redeemStep1` is NOT in the `widgetPage.js` enum (it is a string literal passed inline at `FloatingButton.js:811` and `:816`).

> WARNING: these mount-time members exist only after the floating widget React tree mounts. They are undefined on `window.joyInstance` before mount (e.g. immediately on `joy:ready`). Page-keys other than the four enum values above plus `redeemStep1` are UNVERIFIED.

---

## 2. Per-method detail (non-trivial)

### `customer(url='/info', checkUseMetaField=false)`
`Joy.js:589-595`. Routes `#_tryToReturn(this.#_getData, {field:'customer', url, checkUseMetaField})`. `#_getData` (`Joy.js:301-308`) forwards `checkUseMetaField` straight into `ApiManager.getData({customerId, path:url, checkUseMetaField})` (`Joy.js:302-306`).

`checkUseMetaField` (second param, default `false`) controls a metafield side-load on the GET. When `true`, `ApiManager.getData` appends a query field-selector `m` to the request (`ApiManager.getData` signature `ApiManager.js:81`; logic `ApiManager.js:148-160`):

```js
if (checkUseMetaField) {
  queryObject['m'] = [
    'settings', 'settingsReferral', 'referralProgram', 'shop',
    'tierProgram', 'translation', 'primaryTranslation',
    'settingsBrandingReminder', 'settingWidgetQr', 'allTiers'
  ].filter(key => !!window.AVADA_JOY?.[key]);
}
```

The `m` array is sent only when `checkUseMetaField === true`, and it lists the metafield-derived config keys that ARE ALREADY present on `window.AVADA_JOY` (filter keeps truthy `window.AVADA_JOY[key]`, `ApiManager.js:160`). It tells the server which bootstrap-config fields the client already has, so the response can be scoped accordingly. The customer-accounts widget is the caller that opts in: `apiManager.customer('/info', true)` (`CustomerDashboard.js:163`).

> WARNING: do NOT confuse the filter direction. Source keeps keys where `window.AVADA_JOY[key]` is truthy (`!!window.AVADA_JOY?.[key]`), i.e. fields ALREADY loaded - it is NOT `!window.AVADA_JOY[key]`. Also: if `this.locale !== this.primary` and the locale translation is not cached, `primaryTranslation` is stripped from `m` so it gets refetched (`ApiManager.js:163-173`). Default callers (`customer()` with no args, and `shop()`) pass `checkUseMetaField=false`, so no `m` selector is sent.

```js
const customer = await window.joyInstance.customer();
// customer.point, customer.tierId, customer.referralCode, customer.earn<Event> booleans
```

Returns the `customer` field of GET `/info` (or `url`). If the underlying value is falsey, the promise **rejects** (themes `await` + try/catch or `.catch`). Proxy stub returns `Promise.resolve(null)` on non-SDK plans.

Shape: see captured ground truth [payloads/joy-customer.json](payloads/joy-customer.json). Key fields confirmed there: `point`, `tierId`, `hasTier`, `tierPoint`, `referralCode`, `shopifyCustomerId`, `id`, `email`, `name`, and per-program completion booleans `earnSignUp`, `earnFollowInstagram` (pattern `earn<Event>`). There is NO `completedPrograms` array and NO `signupComplete` field; earned state is the `earn<Event>` booleans (do not invent other completion fields).

### `tiers()`
`Joy.js:679-681`. `#_tryToReturn(this.#_getData, {url: '/tierRewards'})`. No `field`, so it returns the FULL `data` body of GET `/tierRewards` (backend `getAllTiersWithRewards`), not a single picked field. The base tier shape (`name`, `targetPoint`, `systemType`, `contentPerk[]`, `id`, `imageBlock`, ...) matches the `tiers` array in [payloads/joy-programs.json](payloads/joy-programs.json), though `tiers()` hits a different endpoint than `get({path:'/programs'})`. Full captured body: [payloads/joy-tier-rewards.json](payloads/joy-tier-rewards.json).

> WARNING (perk data generations - SKILL.md trap #3): each tier from `/tierRewards` carries BOTH `contentPerk[]` (legacy strings) and `tierRewards[]` (new reward objects `{title, status, showLoyaltyPage, perkVisibility, type, hasExternalLink, ...}`). The new admin "Display configuration" updates only `tierRewards`; `contentPerk` stays frozen at its pre-migration value. Reading `tier.contentPerk` on a migrated shop renders the OLD perk list (confirmed live in [payloads/joy-tier-rewards.json](payloads/joy-tier-rewards.json), where the Gold tier's two fields disagree). Selection rule per the app's ContentPerk.js: legacy only when some tier still ships `contentPerk` AND `AVADA_JOY.tierProgram.isNewSetupContentPerk` is falsy; otherwise `tierRewards` (filter `status !== false`, `showLoyaltyPage !== false`), per tier falling back to whichever model has data. Render rules and per-variant parsers: [theme-patterns.md](theme-patterns.md) section 4.

### `countMembersByTier(cacheKey='')`
`Joy.js:690-708`. `async`, NOT routed through `#_tryToReturn`. Calls `this.#_apiManager.getData({path:'/count-member-tier', customerId:'', cacheKey})`.

```js
const counts = await window.joyInstance.countMembersByTier();
// { "PNdQ0LJHQfIidtbDyqY7": 150, "jYTXUVpHU2F7Xx1c2alD": 75 }  // tierId -> member count
```

Return logic: if `response?.error` -> `#_error(response.error)`. Else `data = response?.data !== undefined ? response.data : response`, returns `data || {}`. Catch -> `#_error(error?.message || 'Failed to count members by tier')`. JSDoc-documented shape is `{ tierId: count, ... }` (`Joy.js:687`).

### `earnPrograms()`
`Joy.js:656-662`. `#_tryToReturn(this.#_getPrograms, {field:'earning', url:'/programs', defaultProgramByType: earningPrograms})`.

`#_getPrograms` (`Joy.js:320-325`) runs `#_getData(params)` and `this.shop()` in parallel, then `programs.filter(x => isPremium(shop) || !defaultProgramByType[x.event]?.proPlan)` - it drops pro-only programs on non-premium shops. Returns the filtered `earning` array. The earning-program object shape (`type`, `event`, `title`, `earnPoint`, `rateMoney`, `earnPointsTiers`, `urlAccount`, `loyaltyPageCustomIcon`, `id`, ...) is the `data.earning[]` shape in [payloads/joy-programs.json](payloads/joy-programs.json). No earning program carries a per-customer `isCompleted` field; completion lives on `customer()` as `earn<Event>` booleans. See [events-and-programs.md](events-and-programs.md), [earn.md](earn.md).

### `redeemPrograms({query={}}={})`
`Joy.js:635-655`. Builds a `URLSearchParams` from `query`: arrays appended per value, objects `JSON.stringify`'d, scalars appended; `undefined`/`null` skipped. Then `#_tryToReturn(this.#_getPrograms, {field:'spending', url:'/programs<queryString>', defaultProgramByType: spendingPrograms})`.

Returns the filtered `spending` array (same pro-filter as `earnPrograms`). The spending-program shape (`redeemType`, `spendPoint`, `event`, `earnAmount`, `isRedeemCheckOut`, `prefix`, `id`, ...) is the `data.spending[]` shape in [payloads/joy-programs.json](payloads/joy-programs.json). The `redeemType` value drives `redeem()` branching below. See [redeem.md](redeem.md).

### `redeem(id, points=0, variantIdSelected)`
`Joy.js:736-742` -> `#_redeemingProgram` (`Joy.js:347-397`). Positional params: `id` (programId), `points` (point spend, only used for dynamic programs), `variantIdSelected`.

Flow:
1. If no `id` -> `#_error('Please enter input')`.
2. Resolve `this.customer()` + `this.redeemPrograms()` in parallel.
3. No customer -> guest flow: `window.joyRegistry?.get('guestRedirectCallback')` called as `callback('login', window.AVADA_JOY?.login_url || '/account/login')`; if it returns truthy, that value is returned; else `#_error('You need to login')`.
4. `programRedeem = programs.find(p => p.id === id)`.
5. `hasCustomPointSpending = programRedeem.redeemType === REDEEM_TYPE_DYNAMIC` (`'dynamic'`). `redeemPoint = hasCustomPointSpending ? parseInt(points) : programRedeem.spendPoint`.
6. If `customer.point < redeemPoint` -> `#_error('<redeemPoint - point> away to use this reward')`. If `redeemPoint <= 0` -> `#_error('Redeem amount must be greater than 0')`.
7. POST `/redeem` body `{locale: window.Shopify.locale||'', shopId, customerId: customer.id, programId: programRedeem.id, variantIdSelected, redeemPoint}`.
8. On `error` -> `#_error(error)`. If `data?.discount?.code`, dispatch `joy:redeemCoupon` `detail:{customer, discount, program: programRedeem, redeemPoint}`.
9. Returns `data` (read `result.data.discount.code` for the coupon).

```js
const result = await window.joyInstance.redeem(program.id, pointsToRedeem);
const code = result?.discount?.code;
```

> WARNING: branching by program type. For `redeemType === 'dynamic'` the caller-supplied `points` is used (`parseInt`); otherwise `program.spendPoint` is used and the `points` arg is ignored. See [redeem.md](redeem.md).

### `revokeCoupon(rewardId)`
`Joy.js:743-745` -> `#_revokeCoupon` (`Joy.js:399-424`). If no `rewardId` -> `#_error('Reward ID is required')`. Reads `shopifyCustomerId = window.AVADA_JOY.customer?.id || ''`; if absent -> `#_error('You need to login')`. POST `/refund-coupon` body `{shopId, rewardId, shopifyCustomerId}`. On `error` -> `#_error(error)`. Dispatches `joy:revokeCoupon` `detail:{customer: AVADA_JOY.customer||{}, reward: data?.reward||{}, couponCode: data?.reward?.couponCode||''}`. Returns `{success:true, data}`.

### `generateLinkReferral(email)`
`Joy.js:749-751` -> `#_generateLinkReferral` (`Joy.js:479-516`). `isLoggedIn = #_isLoggedIn()`. If no `email` and not logged in -> guest-redirect flow, else `#_error('Email required')`. `referrerEmail = email || window.AVADA_JOY.customer.email`. POST `/referral/generateLinkReferral` body `{shopId, email: referrerEmail}`. On `error` -> `#_error(error)`. Dispatches `joy:referralGenerated` `detail:{...data, email: referrerEmail}`. Returns `data`. Proxy stub returns `null` on non-SDK plans. See [events-and-programs.md](events-and-programs.md).

### `sendReferralEmail({refereeEmail, emailCustomer}={})`
`Joy.js:752-754` -> `#_sendReferralEmail` (`Joy.js:518-534`). If no `refereeEmail` -> `#_error('Referee email required')`. `customerEmail = emailCustomer || window.AVADA_JOY?.customer?.email`; if none -> `#_error('Referrer email required')`. POST `/referral/sendEmail` body `{shopId, customerId: customer?.id, emailCustomer: customerEmail, refereeEmail}`. On `error` -> `#_error(error)`. Returns `data`. Proxy stub returns `{success:false, error:'Ultimate plan required'}`.

### `getCouponReferral({emailCustomer, emailFriend}={})`
`Joy.js:755-760` -> `#_getCouponReferral` (`Joy.js:536-550`). `{detectIp, deviceId} = await getBrowserFingerprint()`. POST `/referral/redeemCoupon` body `{emailCustomer, emailFriend, detectIp, deviceId, shopId, presentmentCurrency: window.Shopify?.currency?.active||''}`. Returns `{discountCode: resp.data.couponCodeForFriend}`.

> WARNING: the two guard branches call `this.#_error(...)` WITHOUT `return` (`Joy.js:538`, `Joy.js:548`), so they do NOT short-circuit. A missing `emailFriend` or a server `resp.error` does not stop execution; the method proceeds and will throw on `resp.data.couponCodeForFriend` if `resp.data` is absent. Documented as-is (likely bug).

### `updateDOB(month, day)`
`Joy.js:746-748` -> `#_updateDoB` (`Joy.js:426-473`). If `!month || !day` -> `#_error('Please fill all inputs')`. If `!#_isValidDate({month,day})` -> `#_error('Birthday is invalid')` (`#_isValidDate` allows Feb=29, 30-day months Apr/Jun/Sep/Nov, else 31). Resolve `this.customer()`; if none -> guest-redirect flow, else `#_error('You need to login')`. Resolve `this.earnPrograms()`, find `program.event === EVENT_BIRTHDAY`. Format: if `['ddmm','DD/MM'].includes(program.dateFormat || program.dateType)` -> `DD/MM` (`#_prependZero(day)+'/'+#_prependZero(month)`), else `MM/DD`. POST `/customer/birthday/${customer.id}` body `{birthday: strBirthday, customerId: customer.id, shopId}`. On `resp.error` -> `#_error(resp.error)`, else `{status:true}`.

```js
const res = await window.joyInstance.updateDOB(month, day); // res.status === true on success
```

### `calculateProductPoints({productId, variantId, quantity=1, customerId, locale=''})`
`Joy.js:1049-1083`. `async`, NOT via `#_tryToReturn`. If `!productId || !variantId` -> `#_error('Product ID and Variant ID are required')`. `customerId = customerId || window?.ShopifyAnalytics?.meta?.page?.customerId`. `domain = window.Shopify?.shop || window.location.hostname`. GET `/points-calculator/product?domain=&productId=&variantId=&quantity=[&customerId][&locale]` with `prefix:'/app/api/v1/widgets'`. Returns `response?.data || response`. Catch -> `#_error(error?.message || 'Failed to calculate product points')`. JSDoc data shape (`Joy.js:1047`): `{html, points, pointCurrencyName}` wrapped `{success, data, error}`.

```js
const r = await window.joyInstance.calculateProductPoints({productId, variantId, quantity: 1});
// r.points, r.html, r.pointCurrencyName
```

### `calculateCartPoints({items, autoFetchCart=false, customerId=null, locale=''}={})`
`Joy.js:1094-1148`. `async`, NOT via `#_tryToReturn`. `customerId = customerId || window?.ShopifyAnalytics?.meta?.page?.customerId`. If `items` absent and `autoFetchCart` true: `fetch('/cart.js')`; if `!response.ok` -> `#_error('Failed to fetch cart data')`; reads `cart.items` (`cartItems`) and `cart.total_discount` (`totalDiscount`); catch -> `#_error('Failed to fetch cart: '+msg)`. If still no non-empty `cartItems` array -> `#_error('Cart items are required. Provide items array or set autoFetchCart to true')`. `domain = window.Shopify?.shop || window.location.hostname`. POST `/points-calculator/cart?domain=[&customerId][&locale]` body `{items: cartItems, totalDiscount}` with `prefix:'/app/api/v1/widgets'`. On `error` -> `#_error(error)`, else returns `data`. Catch -> `#_error(error?.message || 'Failed to calculate cart points')`. Each item shape (JSDoc `Joy.js:1088`): `{product_id, variant_id, quantity}`. Return shape `{html, points, pointCurrencyName}`.

### `customerHistory({customerId, before, after, ...query}={})`
`Joy.js:663-670` -> `#_getCustomerHistory` (`Joy.js:327-335`). GET `/activities?<query>` (cursor params `before`/`after`). If no `customerId`, resolves `this.customer()` first; if none -> `#_error('You need to login')`. Returns the raw `getData` response (full body, includes API pagination). Same pattern as `referredHistory` (`/referred`).

### `referralProgram()`
`Joy.js:605-610`. `#_tryToReturn(this.#_getData, {field:'referralProgram', url:'/info'})`. Returns the `referralProgram` field of GET `/info` (referral program config object). Distinct from `settingsReferral()` (referral branding) and from `generateLinkReferral`/`sendReferralEmail`/`getCouponReferral` (referral actions). See [events-and-programs.md](events-and-programs.md).

### `getInfoLoyaltyPage({customerId, hasMyRewardBlock})`
`Joy.js:761-806`. `async`, NOT via `#_tryToReturn`. Computes `detectLocale` via `getDetectLocale(shop, translation)`. Builds `queryFields` from `['earning','spending','tiers','interactWebsiteProgram','settings','primaryTranslation','shop', hasMyRewardBlock && 'rewards','referralProgram','settingsInteractWebsite','settingsReferral']`, filtered (`'rewards'` always kept; `earning`/`spending`/`interactWebsiteProgram`/`tiers` kept only if `window.AVADA_JOY.program[key]`; `settingsInteractWebsite` kept only if `hasSettingsInteractiveWebsite()`; else kept only if `window.AVADA_JOY[key]`). GET `/loyalty-page?m=<fields>` with `customerId`. Returns `{...prepareAvadaJoyData(response), tiers: prepareTiersData({tiers: AVADA_JOY.program.tiers||tiers}), customer: customer||{}, detectLocale, rewards}`.

> WARNING: references bare identifiers `shop` (`Joy.js:763`, `window.AVADA_JOY?.shop || shop`) and `translation` (`Joy.js:764`) that are not declared in the method scope. These throw `ReferenceError` unless those names exist as globals. Documented as-is.

### `getCustomerMilestone({customerId}={})`
`Joy.js:992-994` -> `#_getCustomerMilestone` (`Joy.js:552-583`). If `customerId`: GET `/customer-milestone` for that id; on `error` -> `#_error(error)`, else returns `data`. Else resolve `this.customer()`; if none -> `#_error('You need to login')`; then GET `/customer-milestone` for `customer.id`; on `error` -> `#_error(error)`, else `data`.

### `triggerActivity(actionKey)`
`Joy.js:1001-1037`. `async`. If no `actionKey` -> `#_error('Action key is required')`. Resolve `this.customer()`; if none -> guest-redirect flow, else `#_error('You need to login')`. POST `/trigger-activity` body `{actionKey, shopId, customerId: customer.id}`. On `resp.error` -> `#_error(resp.error)`, else returns `resp` (the FULL response, not `resp.data`). Allowed for Advanced+ plans even under SDK gating (`advancedAllowedMethods`). See [earn.md](earn.md) for custom-trigger programs.

### `hasEarnProgram({customer, program})` - PURE, no API
`Joy.js:815-846`. Returns a boolean (can this customer still earn this program). Branching by `program.event`:

```js
const canEarn = window.joyInstance.hasEarnProgram({customer, program});
```

- `google_maps_review` && `customer.statusGoogleMapsReview === STATUS_REVIEW_APPROVED` -> `false`.
- `birthday`: `false` if `customer.earnBirthDayReward` AND current year === `customer.lastYearGetBirthdayReward`; else `true`.
- `custom_program` + `typeCustom === 'completed_profile'` -> `!customer[getEarnFlag(typeCustom)]`.
- `custom_program` (other) -> `program.canEarn !== false`.
- If `(customer[event] || customer[getEarnFlag(event)])` AND event not a share-social event -> `false` (already earned). This is where the `earn<Event>` booleans on `customer()` ([payloads/joy-customer.json](payloads/joy-customer.json)) gate earning.
- `interact_website` && `!canClaimInteractWebProgram(program, customer)` -> `false`.
- Default: `!(event === EVENT_SIGN_UP && isLoggedIn)` where `isLoggedIn = window.AVADA_JOY.customer?.id`.

Proxy stub returns `false` on non-SDK plans. See [earn.md](earn.md), [events-and-programs.md](events-and-programs.md).

### `handleEarningProgram({program, customer})`
`Joy.js:848-988`. `async`, NOT via `#_tryToReturn`. Heavily branched; executes the earn action for one program.

Guest path (no `customer.id`): try `guestRedirectCallback('login', loginUrl)` (return if truthy). Then by `program`: `custom_program` with `typeCustom` including `visit_page`/`custom_trigger`/`shopify_flow_trigger` and `linkVisit` -> `window.open('https://'+linkVisit,'_blank')` + `{status:true}`; `fill_survey` -> open `urlAtLoyaltyPage || 'https://'+link` + `{status:true}`; else `urlAtLoyaltyPage` -> open it + `{status:true}`; else `#_error('Please login to complete this action')`.

Authenticated path (`customer.id` present): `calculatedEarnFlag = earnFlag || getEarnFlag(event)`.
- `custom_program` visit/trigger/flow types with `linkVisit`: if `visit_page` and host differs from `window.location.origin`, POST `/customer/social?c=<customerId>&p=<programId>` body `{shopId, earnFlag: calculatedEarnFlag, event, customerId}`; then open link -> `{status:true}`.
- `fill_survey` -> open `urlAtLoyaltyPage || 'https://'+link` -> `{status:true}`.
- `urlAtLoyaltyPage` present -> open it -> `{status:true}`.
- `sign_up_newsletter` -> POST `/customer/newsletters` body `{shopId, customerId, shopifyCustomerId, customer:{...customer, acceptsMarketing:true}}`; returns `result.data || result || {status:true}`.
- `interact_website` or a social earning event -> POST `/customer/social?c=<customerId>&p=<programId>` body `{shopId, earnFlag, event, customerId}`; if social and `program.urlAccount`, open the (https-normalized) URL; returns `result.data || {status:true}`.
- Fallback (e.g. birthday / completed-profile needing a form) -> `{status:false, shouldOpenModal:true}` (caller should open a modal).

Proxy stub returns `{success:false, message:'Ultimate plan required'}`. See [earn.md](earn.md).

### `registerRedirectGuest(callback)`
`Joy.js:721-734`. If `callback` is not a function -> warn + return. If no `window.joyRegistry` -> warn + return. Registers `callback` under key `'guestRedirectCallback'`. Read by all guest-gated methods (`redeem`, `updateDOB`, `generateLinkReferral`, `triggerActivity`, `handleEarningProgram`). The callback is invoked `callback('login', redirectUrl)` where `redirectUrl = window.AVADA_JOY?.login_url || '/account/login'`; if it returns truthy, that value short-circuits and is returned to the original caller.

---

## 3. Generic `get()` / `post()` contract

```js
get  = ({path, cacheKey, customerId}) => this.#_apiManager.getData({path, cacheKey, customerId});  // Joy.js:991
post = (path, data) => this.#_apiManager.postData(path, data);                                      // Joy.js:990
```

These are thin pass-throughs to `ApiManager`. They let callers hit ANY endpoint under the default prefix.

### Path prefixing
`ApiManager.getData`/`postData` build the URL as `https://${process.env.API_URL}${prefix}${path}`, default `prefix = '/app/api/v1/popup'` (`ApiManager.js:83`, `ApiManager.js:305`). `get()`/`post()` do NOT accept a `prefix`, so they always hit `/app/api/v1/popup`. `path` must start with `/` (e.g. `'/programs'`, `'/info'`). For GET, query string belongs IN `path` (e.g. `'/rewards?limit=10'`); a `?` already in `path` makes the manager use `&` as the next separator. Only `calculateProductPoints`/`calculateCartPoints` use the `widgets` prefix, and they pass it internally - you cannot reach `widgets` via `get()`/`post()`. See [endpoints.md](endpoints.md) for the full endpoint catalog.

```js
const resp = await window.joyInstance.get({path: '/programs'});
// resp.data.earning[], resp.data.spending[]  (see payloads/joy-programs.json)
const out = await window.joyInstance.post('/some-endpoint', {foo: 'bar'});
```

`get()` return shape mirrors [payloads/joy-programs.json](payloads/joy-programs.json) for `/programs` (`{data:{earning,spending,tiers,tierSettings,interactWebsiteProgram}}`). Signing (HMAC `h`/`ht` on GET, `hashCode`/`hashType` in POST body), the 3s request-registry dedup, and the localStorage minute-cache are all applied by `ApiManager`; see [runtime.md](runtime.md).

### How to detect errors
> WARNING: the transports do NOT check HTTP status. `fetchRequest` (GET) and `makeRequest` (POST) resolve with the parsed JSON body even on 4xx/5xx, and reject only when the body fails `JSON.parse` (`fetchRequest.js`, `makeRequest.js`). A failed request will USUALLY resolve, not throw. Never rely on try/catch alone for API errors.

Detect failure by inspecting the resolved value, in this order:

- `resp === null` - `ApiManager.getData` caught its own error and returned `null` (`ApiManager.js:209-212`). Any GET (including `get()`) can return `null`.
- `resp.error` (string) - set by the server HMAC middleware error branches (e.g. `{error:'Unauthorized request', success:false}`) and by `postData`'s catch (`{error: e.message}`).
- `resp.success === false` - accompanies `error` on HMAC/extension middleware failures. Not every controller sets `success` on success (UNVERIFIED), so treat its absence as non-authoritative.
- `{status:false, error}` - the `#_error` shape returned (NOT thrown) by most `Joy` methods on a client-side guard failure (`Joy.js:199-201`). This is the shape to check after calling typed methods like `redeem`, `updateDOB`, `getCustomerMilestone`, `countMembersByTier`, `calculate*`.
- Promise REJECTION - only the `#_tryToReturn`-routed read getters reject when the inner value is falsey, and they reject with the bare `#_error` method reference (typeof === 'function'), NOT the object `{status:false, error:'No data found'}`. A `.catch`/try-catch handler inspecting `err.error` or `err.status` will find neither (it receives a function); call the reference to get the shape, or just treat any rejection as failure. Cite: `Joy.js:205` (`err(this.#_error)`). So `customer()`, `shop()`, `tiers()`, `earnPrograms()`, `redeemPrograms()`, `rewardList()`, `customerHistory()`, `referredHistory()`, `getCustomerMilestone()` can reject; wrap them in try/catch or `.catch`.

```js
const result = await window.joyInstance.redeem(id, points);
if (result?.status === false || result?.error) {
  // guard failure or server error - result.error has the message
}
```

There is no single normalized envelope; defensively check `resp`, `resp.error`, `resp.success`, `resp.status`, and `resp.data`. See the request-layer detail in [runtime.md](runtime.md).
