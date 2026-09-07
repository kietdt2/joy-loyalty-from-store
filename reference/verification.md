# Verification and report

Two passes. Mechanical first, because a broken page cannot be judged on taste.
Then the finish review against the direction contract.

## Pass 1: widths and states

Screenshot the live preview page, not a local file.

| Width | Check |
|---|---|
| Desktop, as composed | matches the approved Framer layout band by band |
| 390 | the authored mobile layout, not a squeezed desktop |
| 320 | no horizontal overflow anywhere |

States, per SDK band. The design drew some of these; the build must have all
four.

| State | How to reach it | Must show |
|---|---|---|
| Logged out | private window | the join / sign-in path, never an empty shell |
| Loading | throttle, or block the Joy script | a stable skeleton, no layout jump on arrival |
| Empty | a member with no programs or zero balance | the honest empty copy, not `0 undefined` |
| Error | block the Joy endpoint | a readable message and a retry, never a blank band |

Also check:

- signed-in state hides every join / sign-in / create-account CTA, with no flash
  on a cache-served page,
- numbers carry thousands separators,
- keyboard reach through every interactive element, visible focus,
- `prefers-reduced-motion` collapses everything above a fade,
- zero em-dash and zero en-dash-as-separator in rendered text, schema labels,
  defaults and the template JSON,
- console clean of errors originating in the new sections.

## Pass 2: the finish review

Run the `impeccable` finish reviewer against the direction contract, the
approved Framer comp and the chosen world's quality bar. Fix everything material
it returns. Where you disagree, say why in the report rather than dropping it
silently.

Re-check the uniqueness claim yourself, honestly: are the three brand-caused
decisions still visible in the shipped page, or did they get sanded off during
implementation? If they got sanded off, the page failed its brief. Fix it.

## The report

Last message. Complete, honest, and scannable.

```
LOYALTY PAGE - <Brand>

LINKS
  framer preview   <url>       visual design, approved <date>
  live demo        <url>       unpublished dev theme, real data
  theme editor     <url>
  delivery folder  <path>

STORE READ
  <3 lines: what they sell, brand posture, voice>

DESIGN
  bands            <ordered list>
  uniqueness       1. <decision> - <evidence>
                   2. ...
                   3. ...
  tokens           <source of truth, and any contrast adjustment made>
  fonts            <substitutions, if any>

BUILD
  sections         <file -> band, one per line>
  snippets         <...>
  template         <...>
  SDK map          <band -> method, one per line>
  states           <what each SDK band does in all four>

MERCHANT SETUP
  <numbered, exactly what they must do; page creation, Joy config, nav link>

REMAINING
  <every delta, unverified assumption, and thing that degraded; or "none">
```

`REMAINING: none` is only ever written when it is true. If the Joy app was not
installed, if the page record did not exist, if a font was substituted, if a
band could not be verified in a real state, it goes here.
