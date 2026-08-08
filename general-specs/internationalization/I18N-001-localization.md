# I18N-001 — Localization & locale selection

- **Status:** Adopted
- **Group:** Internationalization
- **Applies to:** All user-facing surfaces (web + app).
- **Last updated:** 2026-06-16

## Requirement

1. User-facing copy **MUST** be externalized into a locale catalog, not hard-coded
   inline. Adding a language **MUST NOT** require touching feature code.
2. Apps **MUST** support the agreed baseline locale set: **en, de, fr, es, it,
   zh, ja, ms, id** (English is the fallback). A missing key **MUST** fall back to
   English, never render a raw key or blank.
3. The **user agent MUST auto-detect the user's language** and render in it on
   first contact, with no manual step. On the web that means reading the browser
   `Accept-Language` header (and/or `navigator.language`); in the app it means the
   device locale.
4. Web locale selection **MUST** follow the order: explicit `?lang=` →
   saved cookie → `Accept-Language` header → default (English). An explicit choice
   **SHOULD** be remembered (persisted) so it sticks on later visits.
5. App locale **MUST** follow the device locale by default, with an in-app
   override that persists.
6. Dates, numbers, and pluralization **SHOULD** be formatted per locale.
7. Localized strings are still subject to HTML escaping
   (see [SEC-003](../security/SEC-003-request-integrity-csrf.md)).

## Rationale

A single catalog with a strict fallback keeps nine languages maintainable and
prevents half-translated or broken UI when a key is missing.

## Acceptance criteria

- [ ] All nine locales render with no raw keys or blanks; unknown keys fall back to en.
- [ ] A German-language browser/device gets German UI automatically, with no manual selection.
- [ ] `?lang=de` switches and persists; subsequent visits stay in German.
- [ ] No user-facing string is hard-coded outside the catalog.

## Implementation notes

- **OpenCycle / OpenOutdoor:** `i18n.ts` (9 locales) with `pickLocale()` precedence and a `lang` cookie; marketing + auth pages localized.
- **Emergency / WebhookNotification / Priorize:** locale catalogs (`i18n.tsx` / `strings.ts`) with device + override selection on the app side.
