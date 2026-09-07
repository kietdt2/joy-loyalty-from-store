# Store analysis

Read the storefront the way a brand designer reads it. You are not building a
catalogue dump; you are answering "what would a loyalty page for *this* brand
look like, and why couldn't it belong to anyone else".

## Order of reading

Cheapest signal first. Stop when you can answer every question in the record.

1. **Home page.** The single richest page. Hero posture, section rhythm, what
   they lead with, how loud they are.
2. **A collection page.** Card treatment, grid density, badge language, how
   they handle price and sale.
3. **A product page.** Type ramp at its fullest, button hierarchy, trust
   devices, review UI, cross-sell posture.
4. **Header and footer, on every page.** Nav depth, account entry point,
   newsletter voice, social presence, policy tone.
5. **Any existing rewards / loyalty / account page.** If one exists, it tells
   you what the merchant already promised customers. Match its vocabulary.

## Tooling

Use the Chrome browser tools. Load them in one `ToolSearch` call:

```
select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__read_page,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__get_page_text,mcp__claude-in-chrome__javascript_tool
```

Create a fresh tab; never reuse a tab id from another session.

Screenshot at desktop width and at 390. The mobile pass is not optional: the
loyalty page ships mobile-first, and the merchant's mobile treatment is often
where their real spacing and density decisions show.

Two cheap high-yield probes:

- `/products.json?limit=250` gives catalogue breadth, price band, vendor and
  product-type spread without paging the UI.
- `Shopify.theme` in the page console gives theme name, id and role, which is
  how you resolve the original theme name for the delivery folder.

When either is blocked, fall back to reading the rendered pages and say in the
report that catalogue stats are approximate.

## The store analysis record

Write it into the scratchpad and show it to the user before moving on. Keep it
tight; every line must be able to change a design decision.

```
STORE ANALYSIS - <brand>
origin:            <https://...>
theme:             <name> <version> (source: Shopify.theme | theme folder | inferred)

WHAT THEY SELL
  category:        <one line>
  breadth:         <n products, m collections, dominant product types>
  price band:      <low - high, currency> -> posture: <value | mid | premium | luxury>
  buyer:           <who, in one line, evidenced by copy and imagery>
  purchase rhythm: <one-off | replenishment | seasonal | gifting>   <- drives which loyalty mechanic leads

BRAND
  palette:         <role: hex, from where>
  type:            <heading family / body family, ramp, casing, tracking habits>
  shape language:  <radius, border weight, shadow or none>
  imagery:         <studio | lifestyle | illustrated | UGC; crop and treatment>
  signature device:<the one visual thing that is theirs>   <- must survive into the design
  loudness:        <quiet ... loud, 1-10> and density <sparse ... packed, 1-10>

VOICE
  register:        <formal | warm | playful | technical | irreverent>
  person:          <we/you | brand name | none>
  sample CTAs:     <3 real button strings, quoted>
  sample headings: <3 real headings, quoted>
  vocabulary:      <words they use that you should reuse; words they never use>

LOYALTY CONTEXT
  existing page:   <url or none>
  promised terms:  <any reward vocabulary already in the wild>
  account entry:   <classic accounts | new customer accounts | none visible>
  joy installed:   <yes | no | unknown>   <- probe window.AVADA_JOY / window.joyInstance

INFERENCES MADE
  <every place you guessed because the input did not say, one per line>
```

The `INFERENCES MADE` block is what the user corrects. Never bury a guess in
prose.

## Detecting whether Joy is installed

In the page console:

```js
({ joy: typeof window.joyInstance, avada: !!window.AVADA_JOY,
   points: window.AVADA_JOY?.points, shop: window.AVADA_JOY?.shop })
```

`unknown` is an acceptable answer; a wrong `yes` is not. If Joy is absent, the
page still gets built, every SDK band degrades to its logged-out or empty
state, and the report says so.

## Voice mining

Quote real strings. The loyalty page's headings, tier names and reward labels
are written in the merchant's register, and the only reliable source of that
register is their own copy. Collect at minimum: three CTAs, three section
headings, one paragraph of body, and the newsletter pitch.

If the store is not in English, write the loyalty copy in the store's language
and say so.
