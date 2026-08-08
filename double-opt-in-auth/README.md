# Double Opt-In & Email-Confirmation Flows

Prove a user owns an email address **before** acting on it — by parking the
action, emailing a single-use confirmation link, and only proceeding when the
link is clicked. One token pattern powers every flow: contact/feedback,
password reset, and (new) signup activation.

This is a reusable, parameterized recipe. `threethings` (SST + Hono backend in
`sst/src/`, Remix landing in `sst/landing page/`) is the worked example; swap in
your own table, mailer, and routes to reuse it.

> Status: **partly proven**. The contact-form double opt-in (`/support`) and the
> password-reset flow (`/forgot` + `/reset`) are implemented and in production.
> Signup confirmation, the landing feedback form, and the "Forgot password?"
> link are **designed here and being built**.

## The one pattern behind all flows

```
1. Generate  id (16B) + token (32B), base64url, from a CSPRNG.
2. Store      only sha256(token)  (+ an absolute expiry).  Never the raw token.
3. Email      a link carrying the RAW token (…?token=… [&id=…]).
4. Consume    on click: look up, verify, and invalidate in one shot.
5. Verify     sha256(presented) === stored  via timingSafeEqual; check TTL.
6. Single-use the row/hash is deleted/cleared on success → link works once.
```

The raw token exists only in the email URL. A leaked DB row can't be replayed
(it holds a hash), and a captured link works once and briefly. See
[token-pattern.md](token-pattern.md).

## Why double opt-in

- **Anti-spam / anti-spoof** — the form/endpoint can't be used to fire mail at,
  or register, an address the requester doesn't control.
- **Anti-enumeration** — responses never reveal whether an address is already
  registered (uniform neutral notices + decoy work to flatten timing).
- **Consent (GDPR/CASL)** — a confirmed click is a recorded, provable opt-in
  before you store/process or email the address.

## Folder index

| Area | Doc |
|---|---|
| The reusable confirmation-token recipe | [token-pattern.md](token-pattern.md) |
| Double opt-in **signup** activation (new) | [signup-confirmation.md](signup-confirmation.md) |
| Landing **feedback form** → localized "sent" page (new) | [../support/double-opt-in/feedback-form.md](../support/double-opt-in/feedback-form.md) — moved to [`support/`](../support/) |
| Surface the **"Forgot password?"** link (new) | [forgot-password-link.md](forgot-password-link.md) |
| **Scanner-safe confirmation** — never mutate state on a GET (MUST) | [link-scanner-safe-confirm.md](link-scanner-safe-confirm.md) |
| Edge cases & security checklist | [gotchas.md](gotchas.md) |

## Reference implementations (already in the repo)

- **Contact/support** — `POST /support` parks a `SupportRequest` (with honeypot
  + email validation), emails `…/support/confirm?id=&token=`. `GET /support/confirm`
  is **inert** (renders a "Confirm and send" button); `POST /support/confirm` does
  the atomic fetch-and-delete, constant-time compare, TTL check, then forwards the
  message. GET must never relay — see [link-scanner-safe-confirm.md](link-scanner-safe-confirm.md).
  `sst/src/index.ts`, `sst/src/db.ts` (`putSupportRequest`/`consumeSupportRequest`),
  `sst/src/email.ts` (`sendSupportConfirmEmail`/`sendSupportMessage`).
- **Password reset** — `GET/POST /forgot` + `GET/POST /reset` in
  `sst/src/index.ts`; `sendResetEmail` in `email.ts`; reset hash stored on the
  user (`resetHash`/`resetExpiresAt`), cleared on use.

## Decisions (locked)

- **Token:** `randomBytes(32).toString("base64url")`; store only
  `sha256(token)` (hex); compare with `crypto.timingSafeEqual`.
- **TTLs:** reset **1h**, support **24h**, signup confirm **24h** (proposed).
- **Single-use:** parked rows are deleted atomically on confirm
  (`DeleteCommand … ReturnValues: ALL_OLD`); per-user hashes are cleared on use.
- **Storage:** single DynamoDB table; parked rows carry `expiresAt` in **epoch
  seconds** as the table TTL so unconfirmed rows self-sweep.
- **Anti-enumeration everywhere:** uniform notices + `decoyVerify` to equalize
  timing on the "unknown address" path.
- **Mailer:** SES on Lambda; **local dev logs the link to the console** (no
  verified identity needed) — preserve this fallback in every new email fn.
- **Landing confirmation pages** are localized in all 8 i18n languages
  (en/de/zh/ms/id/fr/es/pl).

## Open items

1. Signup confirmation changes login semantics — decide whether existing
   (pre-feature) accounts are grandfathered as confirmed (recommended: yes, via
   `confirmedAt` absent ⇒ treat legacy `createdAt < cutoff` as confirmed).
2. Resend-confirmation endpoint + its own rate-limit bucket.
3. Where the signup confirmation link lands (hosted API page vs. Remix route).
4. SES must be out of the sandbox to mail arbitrary recipients in prod.
