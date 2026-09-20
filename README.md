# NFRs & Repetitive Specs

Cross-cutting **non-functional requirements** and the **recipes we'd otherwise
rewrite in every app** — stated once, reused, instead of copy-pasted and drifted.

## Canonical specs

**[`general-specs/`](general-specs/)** — 56 numbered specs, RFC 2119 keywords,
each with status · applicability · requirement · rationale · acceptance criteria.
→ **[INDEX.md](general-specs/INDEX.md)** lists every one.

| Group | Prefix | |
|---|---|---|
| Public pages & routes | `PAGE` | [pages/](general-specs/pages/) |
| UI & design | `UI` | [design/](general-specs/design/) · [ui/](general-specs/ui/) |
| Authentication | `AUTH` | [authentication/](general-specs/authentication/) |
| Security | `SEC` | [security/](general-specs/security/) |
| Privacy | `PRIV` | [privacy/](general-specs/privacy/) |
| Reliability | `REL` | [reliability/](general-specs/reliability/) |
| Sync | `SYNC` | [sync/](general-specs/sync/) |
| Data & API | `DATA` | [data-and-api/](general-specs/data-and-api/) |
| Delivery | `DEL` | [delivery/](general-specs/delivery/) |
| SEO | `SEO` | [seo/](general-specs/seo/) |
| i18n · a11y | `I18N` `A11Y` | [internationalization/](general-specs/internationalization/) · [accessibility/](general-specs/accessibility/) |

## Feature bundles

Every requirement bearing on **one surface**, assembled so you can build it
without hunting. Specs are copies; `general-specs/` stays authoritative.

- **[`support/`](support/README.md)** — the support/contact form: page, `POST /support`, double-opt-in mail, "sent" page. Route spec: [PAGE-003](general-specs/pages/PAGE-003-support-route.md).
- **[`privacy-page/`](privacy-page/README.md)** — the `/privacy` route: notice, links pointing at it, imprint, user-rights actions. Route spec: [PAGE-001](general-specs/pages/PAGE-001-privacy-route.md).

## Playbooks

Worked recipes for what bites once per app.

- **[`IAP-Subscriptions/`](IAP-Subscriptions/README.md)** — **in-app purchases only**, end to end. Numbered `IAP-NNN` specs ([IAP-001](IAP-Subscriptions/specs/IAP-001-purchase-progress-feedback.md) — purchase progress & outcome feedback; [IAP-002](IAP-Subscriptions/specs/IAP-002-price-display-fidelity.md) — price display fidelity) plus the playbooks:
  - [`app-store-iap-setup/`](IAP-Subscriptions/app-store-iap-setup/README.md) — API-driven IAP setup (App Store Connect + RevenueCat).
  - [`revenuecat-integration/`](IAP-Subscriptions/revenuecat-integration/README.md) — entitlement sync, webhooks, the multi-app trap.
  - [`storekit-paywall-gating/`](IAP-Subscriptions/storekit-paywall-gating/README.md) — macOS client-side gating & sandbox testing.
  - [`non-tracking-purchases/`](IAP-Subscriptions/non-tracking-purchases/README.md) — ATT vs. privacy label (Guideline 5.1.2(i)).
- **[`general-specs-payment/`](general-specs-payment/README.md)** — **web/card payment** spec: providers, plans, billing API. (Store IAP lives in `IAP-Subscriptions/`.)
- **[`double-opt-in-auth/`](double-opt-in-auth/README.md)** — email-confirmation flows; confirm links must be inert on GET. Route specs: [PAGE-003](general-specs/pages/PAGE-003-support-route.md), [PAGE-004](general-specs/pages/PAGE-004-password-reset-routes.md).
- **[`deployment-self-test-and-e2e/`](deployment-self-test-and-e2e/README.md)** — per-stage auth verification, `/tests`, Playwright.
- **[`scaffold-specs/`](scaffold-specs/)** — reusable prompts (product brainstorm, app scaffold, dotenv).

## Conventions

- One spec, one file, stable `GROUP-NNN` ID — **never reused**, even after retirement.
- It belongs here if it would otherwise be copy-pasted into **three or more apps**; app-specific behaviour stays in that app's `specs/`.
- Bundles **copy, don't fork**: fix `general-specs/`, then refresh the copies — never the reverse.
- An `Adopted` spec an app doesn't meet is a **gap to track**, not a reason to soften the spec.
