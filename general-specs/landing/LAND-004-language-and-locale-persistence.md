# LAND-004 — Language switcher, locale persistence & cache safety

- **Status:** Proposed
- **Group:** Landing page
- **Applies to:** Every public web surface that renders more than one language at
  the same URL.
- **Last updated:** 2026-10-04

[I18N-001](../internationalization/I18N-001-localization.md) sets the locale
order (`?lang=` → cookie → `Accept-Language` → English). This spec is the web
mechanics underneath it: the switcher control, the cookie, the redirect, and the
CDN cache that silently breaks all of it.

## Requirement

### The switcher

1. A language switcher **MUST** be present in the header on every public page,
   including the legal pages, and in the mobile drawer
   ([LAND-003](LAND-003-responsive-navigation.md)).
2. Each option **MUST** be labelled in its **own** language ("Deutsch",
   "Français", "日本語"), never in the current page's language, so a visitor who
   can't read the current page can still find theirs.
3. The switcher **MUST** be a real `<form method="post" action="/set-locale">`
   with a visible submit button, so it works as a plain document POST without
   JavaScript.
4. **Keyboard-safe auto-submit.** Choosing with a pointer or touch **MAY** submit
   immediately. Arrowing through the options with the keyboard **MUST NOT**
   submit on every change. The keyboard user submits with Enter or the button.
   Otherwise each arrow press navigates away before they reach their language.

### Persistence

5. `POST /set-locale` **MUST** do the following:
   - validate the locale against the supported list, falling back to the default;
   - set a `locale` cookie (`Path=/`, `HttpOnly`, `SameSite=Lax`, `Secure`,
     about 1 year);
   - redirect `303` back to the page the visitor came from.
   The redirect target **MUST** be a same-site path: it starts with `/`, does not
   start with `//`, and carries no scheme or host. Anything else goes to `/`. An
   open redirect here is a phishing primitive on our own domain.
6. `GET /set-locale` changes nothing and redirects to `/`.

### Mixed-language pages

7. A page is either **fully localized** or **explicitly English-only**. An
   English-only page (EULA, security overview, technical docs) **MUST** declare
   `lang="en"` on its content root, whatever the visitor's locale, and **SHOULD**
   say once, in the visitor's language, that the page is available in English
   only. A localized page **MUST NOT** contain hard-coded English sections. The
   privacy notice and the imprint are legal obligations and **MUST** be
   localized ([PAGE-001](../pages/PAGE-001-privacy-route.md)).

### Cache safety

8. **If the language is chosen per request at one URL, the HTML MUST NOT be
   shared-cached without the locale in the cache key.** Do one of the following:
   - send `Cache-Control: private, no-store` and `Vary: Cookie` on every HTML
     document;
   - put the locale cookie (and `Accept-Language`, normalized) into the CDN
     cache key;
   - move to per-locale URLs (`/de/…`), which also unlocks `hreflang`
     ([SEO-003](../seo/SEO-003-metadata-and-social-cards.md) §9).
   Static assets (JS, CSS, images) stay long-cached and immutable.
9. The rendered `<html lang>` and `og:locale` **MUST** match the language the
   body was actually rendered in ([SEO-003](../seo/SEO-003-metadata-and-social-cards.md) §7).

## Rationale

The cache rule (§8) is the one that cost real time. The CDN's default policy left
cookies out of the cache key, so the first visitor's language was cached and
served to everyone. Switching to English then "did nothing", because the German
copy cached first kept coming back. Nothing in the app was wrong; the CDN was.
Every app that picks the language per request at one URL has the same trap
waiting.

The other rules come from equally concrete failures. They are: a dropdown that
navigates on the first arrow press, an open redirect hiding in a `redirectTo`
field, and a page whose hero speaks German while its feature sections speak
English.

## Acceptance criteria

- [ ] Every public page, legal pages included, renders the switcher, and so does the mobile drawer.
- [ ] Each option is labelled in its own language.
- [ ] With JavaScript disabled, choosing a language and pressing the button switches the page.
- [ ] Arrowing through the options with the keyboard does not navigate until Enter.
- [ ] `POST /set-locale` with `redirectTo=//evil.example` or `https://evil.example` lands on `/`.
- [ ] The cookie is `HttpOnly`, `Secure`, `SameSite=Lax`.
- [ ] Two clients with different locales, fetching the same URL through the CDN back to back, each get their own language.
- [ ] English-only pages declare `lang="en"`. Localized pages contain no hard-coded English section.
- [ ] `<html lang>` and `og:locale` match the rendered body.

## Implementation notes

- **WebhookPush** is the reference: `components/LanguageSwitcher.tsx` (22
  locales, native labels, pointer-only auto-submit, always-visible apply
  button), `routes/set-locale.tsx` (`safeRedirect`), `locale.server.ts` (cookie,
  1 year) and `i18n.ts` `detectLocale` (q-ranked `Accept-Language`, `zh-*` → `zh`).
- **Cache (§8):** `root.tsx` exports `headers` with `Cache-Control: private,
  no-store` and `Vary: Cookie` for every document, because the SST Remix
  CloudFront cache policy excludes cookies from the key. The trade-off is that
  no HTML is cached at the edge. Per-locale URLs would recover it, and are the
  open item behind SEO-003 §9.
- **Gaps:** the cookie is not marked `Secure`, and the redirect is a `302`, not
  a `303` (§5). `/terms`, `/security`, `/ai` and `/docs` are English-only by
  design but inherit the visitor's `lang` (§7), and the landing page mixes
  languages (see [LAND-001](LAND-001-landing-page-anatomy.md) §7).
