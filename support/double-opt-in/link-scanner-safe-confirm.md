# NFR: confirmation links must be scanner-safe (no state change on GET)

**Requirement (MUST).** Any action triggered by following an emailed link —
forwarding a message, activating an account, marking an email verified, applying
a reset — MUST NOT happen on the `GET` that the link resolves to. The `GET` MUST
be side-effect-free and render a page with a **button that POSTs** to a separate
handler; only that `POST` performs the state change.

This is the correction to the original reference design in this folder, which had
`GET /support/confirm` forward the message directly. That design is **retracted**
— see "Corrections" below.

## Why (threat model)

The emailed link is delivered to a **user-supplied** address. Corporate inbound
mail security (Microsoft Defender **SafeLinks**, Proofpoint **URL Defense**,
Mimecast, Barracuda) **automatically issues a GET to every URL in inbound mail**
to scan it for malware — before, and independent of, any human. If the `GET`
carries out the action, the scanner completes it on the recipient's behalf.

Observed impact (priorize, July 2026): a bot submitted the contact form with
**harvested business addresses** (real orgs: `*.gov`, consultancies, distributors)
as the sender plus spam text. Each org's link-scanner auto-fetched the "confirm
your message" link, relaying the spam to `info@sp33c.tech` — the double opt-in was
fully defeated because a *GET* could confirm. Dozens of
`threethings message from <harvested-address>` emails resulted.

The same class of bug exists wherever a GET mutates state:
- account **email verification** (`GET /verify?email&code`) → scanner auto-verifies,
- signup **activation**, **unsubscribe**, reset **apply** — any GET side effect.

## Acceptance criteria

1. **GET is inert.** The confirm/verify/activate GET performs no DB write and
   sends no mail. It reads only the opaque `id`/`token` from the query and
   renders an HTML page. A `curl` of the link (any number of times) changes
   nothing server-side.
2. **POST does the work.** A sibling `POST` handler consumes the token
   (atomic single-use), does the constant-time compare + TTL check
   (see [gotchas.md](gotchas.md)), then performs the action.
3. **The button is a plain form**, not JS-auto-submitted (`<form method="POST">`
   + `<button>`), so scanners that only follow links — and don't submit forms —
   can't trip it, and the page still works with JS disabled.
4. **Rate-limit the POST** by IP (the GET may stay unlimited since it's inert).

```ts
// GET: inert — just render the button (id/token flow straight through to the form)
app.get("/support/confirm", (c) => {
  const id = String(c.req.query("id") ?? "");
  const token = String(c.req.query("token") ?? "");
  if (!id || !token) return c.html(renderSupportPage({ error: "…invalid…" }));
  return c.html(renderSupportConfirmPrompt({ id, token })); // <form POST> + button
});

// POST: the only place the message is relayed / the state is mutated
app.post("/support/confirm", async (c) => {
  const form = await c.req.parseBody();
  const id = String(form.id ?? ""), token = String(form.token ?? "");
  // rate-limit(IP) → consumeSupportRequest(id) → timingSafeEqual → TTL → send
});
```

## Companion input-hardening (public/semi-public submit endpoints)

Where a form feeds an email path, also apply on the **submit** handler:

- **Honeypot.** A hidden field (e.g. `website`) no human sees; if non-empty,
  silently accept (render the normal "sent" page) and drop. Add it to **every**
  rendering of the form — server-rendered pages *and* the landing form.
- **Real email validation**, not `email.includes("@")`. Use a shared
  `emailOk` regex (`/^[^\s@]+@[^\s@]+\.[^\s@]+$/`, len ≤ 254) **and** reject
  header-injection chars `[\r\n,;<>]` before the value is used as an SES
  `Reply-To` / `To`.
- Keep the existing **rate limits** (by IP and IP+email); they slow but don't
  stop an IP-rotating botnet, so they are a floor, not the fix.
- (Optional, stronger prevention) Cloudflare **Turnstile** on the submit form —
  deferred where it needs an external account/keys.

## Applies to

| Project | Flow | Status |
|---|---|---|
| **priorize / threethings** | `GET /support/confirm` relayed message | **Fixed + deployed** (`sst deploy --stage prod`, Jul 2026): GET→button, new `POST /support/confirm`, honeypot, email validation. |
| **webhook / WebhookPush** | `GET /verify` auto-marks `emailVerified`; `POST /feedback` lacks email-validation + honeypot | **Open** — same GET-side-effect class (verify), plus `/feedback` input hardening. |
| **timewarp / timelogger** | none — contact is `mailto:`, auth emails via Cognito (codes, POST-verified) | **N/A** — no relay/GET-confirm to harden. If a support form is added later, apply this NFR. |

## Reference implementation

priorize commit `e5b7a84` — `sst/src/index.ts` (`GET`/`POST /support/confirm`,
honeypot + `validEmail` on `POST /support`), `sst/src/page.ts`
(`renderSupportConfirmPrompt`, honeypot field), `sst/landing page/app/routes/_index.tsx`
(landing-form honeypot).
