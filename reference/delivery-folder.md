# Delivery folder

One folder the merchant can diff, push, or hand to their own developer. It holds
their original theme untouched plus everything you added.

## Naming

```
[Joy-Loyalty-Team] <Original Theme Name>
```

Exact, including the brackets and the single space after them. Resolve
`<Original Theme Name>` in this order:

1. **The live theme's name, from `shopify theme open --store <store>`**, which
   step 0 has already run. This is the merchant's own name for the theme as it
   appears in their admin, and it outranks every other source: it is what they
   will look for when the delivery lands.
2. `name` in the theme folder's `config/settings_schema.json`, first block
   (`theme_name`). This is the *theme vendor's* name, so it says `Sense` where
   the merchant's admin may say `Sense — live Feb 2026`. Good when no store is
   reachable; second-best when one is.
3. `Shopify.theme.name` read from the live storefront console.
4. The `theme_info` block or the export folder's own name.

An export folder's filename is the worst of these — it carries dates, export
timestamps and whatever the person who downloaded it typed. Use it only when
nothing else is available, and say in the report that you did.

Use the human name, spaces and capitals intact: `Dawn`, `Impulse`,
`Prestige`. Do not kebab-case it, do not append a version, do not add a date.
If the name contains characters the filesystem rejects, strip only those and say
what you stripped.

Create it as a sibling of the source theme folder unless the user names a
location.

## Contents

```
[Joy-Loyalty-Team] <Original Theme Name>/
  <every file of the original theme, byte-identical>
  sections/
    joy-<prefix>-*.liquid          <- new
  snippets/
    joy-<prefix>-tokens.liquid     <- new, the locked token sheet
    joy-<prefix>-*.liquid          <- new, if the build needed shared partials
  templates/
    page.joy-loyalty-page.json     <- new, the loyalty page template
  assets/
    joy-<prefix>-*                 <- new, only if the design needs real assets
  JOY-LOYALTY-README.md            <- new, the handover note
```

## Naming the template

**`templates/page.joy-loyalty-page.json`.** Not `page.rewards.json`, not
`page.loyalty.json`.

The template name is what a merchant reads in the theme editor's template
dropdown, next to every other page template their theme ships. `rewards` tells
them nothing about where it came from; `joy-loyalty-page` says exactly which
build owns it, which matters when someone opens the store a year later.

The template name and the **page handle are independent**. Keep the handle short
and customer-facing, `/pages/rewards`, and let the template carry the
identifying name. Say this in the README, because the two being different
surprises people.

## Rules

- **Copy the original in first, and verify it is unchanged.** `cp -R` the whole
  theme, then confirm nothing was modified before you add anything. The merchant
  must be able to diff the folder against their export and see only additions.
- **Renaming a template leaves the old one on the remote theme.** `--nodelete`
  protects the merchant's files, so it also protects your stale ones. After a
  rename, push the old path once without `--nodelete` to clear it, and verify by
  pulling `templates/page.*.json` back. Two loyalty templates in the dropdown is
  worse than none.
- **Additions only.** Never edit an existing theme file. If a new section needs
  a global style or a script, it carries it itself, scoped to
  `.section-{{ section.id }}`.
- **One exception, and it needs permission:** adding the page template requires
  no edit, but linking the page into the nav does. Do not touch
  `config/settings_data.json` or any menu. Tell the merchant to add the nav link
  themselves.
- **No secrets.** No API keys, tokens, `.env`, or CLI credentials in the folder.
- **New files only under `joy-<prefix>-`** so every addition is greppable and
  removable in one command.

## The handover note

`JOY-LOYALTY-README.md`, written for the merchant's developer, not for you:

```markdown
# Joy loyalty page - <Brand>

Built by the Joy Loyalty Team on top of <Original Theme Name>, unmodified.

## What was added
<table: file, what it is, which band>

## How to see it
1. `shopify theme push --unpublished` from this folder, or upload the zip.
2. Open the preview URL. The page lives at /pages/<handle>.
3. Customise every band in the theme editor; all settings are exposed in schema.

## Merchant setup required
- Create a page with handle `<handle>` and assign the `<template>` template.
- <Joy app install / program config, if not already done>
- <nav link, which we did not add>

## Notes
<font substitutions, contrast adjustments, anything degraded, anything unverified>
```

Keep it honest. Anything you could not verify goes in `Notes`, not in silence.
