# Dev theme and the live demo URL

The deliverable includes a URL the merchant clicks and sees the real page, with
real Joy data where Joy is installed. It is always an **unpublished** theme.

## Auth

You cannot log in for the user. Check first:

```bash
shopify version && shopify auth logout --help >/dev/null 2>&1; echo "cli present"
```

If not authenticated, tell them to run it themselves so the output lands here:

```
! shopify auth login --store <store>.myshopify.com
```

Wait for them. Do not attempt a browser OAuth flow through the automation tools,
and never handle their credentials.

If Shopify CLI is unavailable entirely, fall back: zip the delivery folder, tell
them to upload it under Online Store > Themes > Add theme > Upload zip, and say
clearly in the report that the live demo URL is pending their upload.

## Open the store first, before creating anything

As soon as you have the store URL — at the start of the job, not at push time —
run:

```bash
shopify theme open --store <store>.myshopify.com
```

Do it before authoring the delivery folder, and certainly before the first push.
It costs one command and settles four things that are expensive to discover
late:

- **Auth, on this specific store.** `shopify auth login` succeeds against a
  partner account that has no access to *this* shop. `theme open` fails loudly
  and immediately, rather than at the end of a build.
- **The store resolves at all.** A typo'd handle, a store that was renamed, or
  a URL that is actually a custom domain pointing somewhere else all surface
  here.
- **Which theme is live, and what it is called.** This is where the delivery
  folder's name comes from — `[Joy-Loyalty-Team] <Original Theme Name>` — and
  guessing it from the export folder's filename is how it ends up wrong.
- **The real storefront in front of you**, which is step 1 of the workflow
  anyway.

It also lists the shop's existing themes, which tells you whether a previous
Joy delivery is already there. That is the difference between a second theme
nobody asked for and an update to the one they are already reviewing.

Only after this do you create the theme and name it.

## Push

The **first** push creates the theme, from inside the delivery folder:

```bash
shopify theme push --unpublished --theme "Joy Loyalty - <Brand> <yyyy-mm-dd>" --store <store>.myshopify.com
```

- `--unpublished` is non-negotiable. Never `--live`, never `--allow-live`,
  never `theme publish`.
- `shopify theme dev` is fine for your own iteration, but its URL is local and
  dies with the process. The deliverable is the unpublished theme's preview URL.
- Date the theme name so repeat deliveries do not collide.
- Shops cap at 20 themes. If the push fails on that limit, ask which theme to
  reuse rather than deleting one yourself.

## Every update: pull the whole theme, edit, push scoped

The remote theme is the source of truth, not your folder. Between your pushes it
changes for reasons that have nothing to do with you:

- the merchant edits a section or a setting in the theme editor,
- **an app renames its handle and Shopify rewrites every block reference**,
- Shopify reformats JSON templates it has touched.

Any of those are lost the moment you push a stale local copy. So the loop is
always three steps, run automatically, without being asked:

```bash
# 1. Snapshot, so you can see what the pull brought back
cp -R "<delivery folder>" /tmp/theme-before-pull

# 2. Pull the WHOLE theme, not just settings
shopify theme pull --theme <id> --store <store>

# 3. Make your edits, then push only the files you author
shopify theme push --theme <id> --store <store> --nodelete \
  --only "sections/joy-<prefix>-*.liquid" \
  --only "snippets/joy-<prefix>-*.liquid" \
  --only "templates/page.joy-loyalty-page.json" \
  --only "assets/joy-<prefix>-*"
```

`--only` restricts the upload. `--nodelete` stops the CLI deleting remote files
absent from your folder. This is why the naming convention earns its keep: one
prefix makes the whole scope expressible in four globs.

### Read the diff, do not skim it

`diff -rq` between the snapshot and the pulled folder will report dozens of
files. Most are noise; a few are decisions. Classify before you continue:

```bash
diff -rq /tmp/theme-before-pull "<delivery folder>"
```

- **JSON templates almost always differ cosmetically.** Shopify returns them
  pretty-printed with a `/* auto-generated */` header, so a byte diff is
  meaningless. Strip comments and compare parsed objects to find real changes.
- **App handle renames appear as mass changes.** Seeing the same
  `shopify://apps/<old>/...` to `shopify://apps/<new>/...` swap across 80 files
  is Shopify repointing an app, not the merchant working. Keep it, and say so.
- **A change inside your own section is the merchant editing your work.** Keep
  it. If it belongs in the stylesheet rather than an inline style, move it there
  and say you did, but never silently revert it.
- **`config/markets.json` disappears on pull.** Shopify does not serve it back.
  Restore it from the snapshot or the folder loses a file it should keep.

If a pulled change conflicts with an edit you were about to make, name the
conflict and ask. Their editor change outranks your local copy.

Note that `settings_data.json` and pulled JSON templates carry a `/* ... */`
header, so they are not parseable as strict JSON. Strip comments before diffing.

Put the pull and push commands in the handover README. The merchant's developer
will otherwise reach for the bare push and hit the same trap.

Capture from the output:

```
preview  https://<store>.myshopify.com/?preview_theme_id=<id>
editor   https://admin.shopify.com/store/<store>/themes/<id>/editor
page     https://<store>.myshopify.com/pages/<handle>?preview_theme_id=<id>
```

The **page** URL is what you send. A bare preview URL lands on the home page and
the merchant has to hunt.

## The page must exist

A template alone renders nothing. The page record is created in admin, and you
usually cannot create it. Either:

- the merchant creates a page with the handle and assigns the template, per the
  handover note, or
- if you have Admin API access they explicitly granted, create it and say you
  did.

Until the page exists, the demo URL 404s. Check before sending: open the page
URL in the browser and confirm it renders. Sending an untested URL is the most
common way this stage fails.

**The quieter failure is a page that exists but is not assigned, and it returns
200.** If the merchant already runs a Joy loyalty page, the handle resolves,
the theme is correct, and the page renders the *old* app-block layout from
`templates/page.json` while your sections sit unused. Nothing errors. Detect it
by grepping the fetched HTML for one of your own class names rather than by
eyeballing a screenshot:

```bash
curl -sL "https://<domain>/pages/<handle>?preview_theme_id=<id>" \
  | grep -c "joy-section--"      # 0 means the template is not assigned
```

Zero hits with a 200 status means the assignment step has not happened. Say that
explicitly rather than reporting the demo as live, and do **not** reach for
`templates/page.json` to force a render: it is the default template for every
page on the theme, so the whole storefront becomes the loyalty page.

Assignment needs Admin API `write_content`, which the Theme CLI does not hold,
so ask for a token up front if the demo URL is part of the deliverable. Without
one, the honest report is: sections pushed and schema-verified, visual pass
pending assignment, plus the exact click path (Content > Pages > the page >
Theme template > `joy-loyalty-page`).

## Verify before sending

Open the page URL in a fresh tab and confirm:

- every band renders, in the approved order,
- no console errors from the new sections,
- SDK bands show data, or their honest logged-out / empty state,
- the theme editor lists every `Joy: ...` section with working settings.

Then hand over both URLs with one line each on what they are for, and state
plainly that the theme is unpublished and their live store is untouched.
