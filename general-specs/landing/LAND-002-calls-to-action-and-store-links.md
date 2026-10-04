# LAND-002 — Calls to action & store links

- **Status:** Proposed
- **Group:** Landing page
- **Applies to:** Every public web page that links to an app store, a sign-up
  flow, or a pricing anchor: landing page, nav, docs, security, emails.
- **Last updated:** 2026-10-04

## Requirement

1. **One constant per external destination.** The App Store URL, Play URL,
   TestFlight URL and web sign-up URL **MUST** each be defined once
   (`APP_STORE_URL`, `PLAY_URL`, …) and imported by every component that links
   them: hero, nav, drawer, footer, docs. Two literals for the same store link
   **MUST NOT** exist.
2. **No placeholder ships.** A build **MUST** fail if a page bundle contains a
   placeholder store link (`REPLACE_ME`, `idXXXX`, `TODO`, an empty `href`). An
   app that is not live yet hides the store CTA, or shows "Coming soon"
   ([LAND-012](LAND-012-truthful-marketing-claims.md)). It **MUST NOT** link to
   a dead listing.
3. **The hero has at most three CTAs, ranked.** There is one primary action,
   visually dominant (download, or start free), and at most two secondary ones
   (start free / see pricing). Each CTA **MUST** be a real `<a href>` to a real
   destination: a store, `/signup` or `/#pricing`.
4. **Store badges follow the store's rules.** App Store and Google Play CTAs
   **MUST** use the store's official badge artwork, or its exact wording
   ("Download on the App Store", "Get it on Google Play") with the official
   logo. They **MUST NOT** be restyled into something the store's marketing
   guidelines forbid, and **SHOULD** be localized where the store provides
   localized badges.
5. **Store links open the listing directly.** Store links use the canonical
   listing URL (`https://apps.apple.com/app/id<ID>`), not a search link or a
   redirect through a tracking domain, and **MUST NOT** carry tracking
   parameters that identify the visitor
   ([PRIV-003](../privacy/PRIV-003-tracking-consent-att.md)).
6. **Fine print under a CTA is a claim.** Text such as "no credit card · cancel
   anytime" or "free to start" **MUST** be true of the shipped sign-up and
   purchase flow ([LAND-012](LAND-012-truthful-marketing-claims.md)).
7. **The final CTA repeats the primary action**, worded as an outcome
   ("Start free"), with one button.
8. **External links are marked and safe.** Links that leave the site and open a
   new tab carry `rel="noreferrer"` (which implies `noopener`) and are announced
   as external, for example with an icon that has an accessible name.

## Rationale

The store link is the most valuable link on the page, and it is the one most
likely to drift. It is written once in the hero, again in the nav, again in the
drawer, and each copy is updated independently when the app goes live. One
constant plus a build-time placeholder check turns "the nav still links to
`idREPLACE_ME`" from a launch-day embarrassment into a failed build.

## Acceptance criteria

- [ ] Grep finds exactly one definition of each store URL, and every store link imports it.
- [ ] The production build fails when a placeholder store link is present.
- [ ] The hero shows one primary CTA and at most two secondary CTAs, all real links.
- [ ] Store CTAs use official badge artwork or the exact official wording.
- [ ] Store links carry no visitor-identifying parameters.
- [ ] Every claim printed under a CTA is true of the shipped flow.
- [ ] External new-tab links have `rel="noreferrer"`.

## Implementation notes

- **WebhookPush:** the hero CTAs are "Download on the App Store" (Apple logo,
  primary), "Start free" → `/signup` and "See pricing" → `#pricing`. The hero
  note reads "no credit card · cancel anytime", and the final CTA is one button
  to `/signup`.
- **Gap (§1, §2):** the hero links `https://apps.apple.com/app/id6787688132`,
  while `LandingNav.tsx` defines its own `APPSTORE_URL =
  "https://apps.apple.com/app/idREPLACE_ME"` for the drawer entry. That is two
  literals, and one of them is a shipped placeholder. `marketing/RECONCILE.md` §3
  also lists a TestFlight `join/REPLACE_ME`.
