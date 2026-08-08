# Landing feedback form (double opt-in → localized "sent")

**Goal:** a feedback form on the Remix landing page. On submit, the user gets a
confirmation email; clicking the link forwards the message to support and lands
them on a page that says **"your message has been sent"** in their language.

Reuse the **existing** backend `/support` flow — do not build a second one.

## Backend: already done

- `POST /support` (form-encoded `email`, `message`, `kind`, `meta`) parks the
  message and emails `…/support/confirm?id=&token=`. It also runs a honeypot +
  real email validation (see [link-scanner-safe-confirm.md](link-scanner-safe-confirm.md)).
- `GET /support/confirm` is **inert** — it only renders a page with a
  "Confirm and send" button. `POST /support/confirm` validates + forwards via
  `sendSupportMessage` (Reply-To = submitter) and renders a "message sent" page.

> ⚠️ **Do not relay on the GET.** The earlier design forwarded the message
> directly from `GET /support/confirm`; email URL-scanners auto-fetched the link
> and relayed spam. Confirmation MUST be a POST — see
> [link-scanner-safe-confirm.md](link-scanner-safe-confirm.md).

Both live on the Auth function (`AUTH_BASE_URL`). The Remix landing already
exposes `authBase` from its loader (see `_index.tsx`).

## Two integration options

**A. Post straight to the backend (simplest).** Render a plain
`<form method="POST" action="{authBase}/support">` on the landing page with
`email` / `kind` / `message`. The browser navigates to the API's HTML
"check your email" page, and the confirm link lands on the API's HTML "sent"
page. Zero new backend code; the localized landing page is bypassed for the two
result screens.

**B. Localized result page on the landing (matches the request).** Keep the form
on the landing; after confirm, land the user **back on a Remix route** showing a
localized "message sent". Make `/support/confirm` redirect to the landing:

```ts
// at the end of POST /support/confirm, on success (never the GET):
const back = process.env.LANDING_BASE_URL; // already an iosapp env; add to the Auth fn env
return back ? c.redirect(`${back}/?sent=1`) : c.html(renderSupportSentPage(email));
```

Then in the landing `_index.tsx` loader, read `?sent=1` and surface a localized
banner, or add a dedicated route `routes/sent.tsx`. Recommended: a banner on the
index so the user sees it in context.

> Recommendation: **B**, since the request explicitly wants the confirmation to
> land on a localized "message has been sent" page.

## i18n keys to add (`app/i18n.ts`, all 8 langs)

```
feedbackTitle      e.g. "Send feedback"
feedbackEmail      "Your email"
feedbackMessage    "Your message"
feedbackKindBug    "Bug"  / feedbackKindIdea "Feedback"
feedbackSubmit     "Send"
feedbackCheckMail  "Almost there — check your email to confirm and send."
feedbackSent       "Your message has been sent. Thank you!"
feedbackError      "Something went wrong — please try again."
```

Suggested translations for `feedbackSent`:

| lang | text |
|---|---|
| en | Your message has been sent. Thank you! |
| de | Deine Nachricht wurde gesendet. Vielen Dank! |
| zh | 您的消息已发送。谢谢！ |
| ms | Mesej anda telah dihantar. Terima kasih! |
| id | Pesan Anda telah terkirim. Terima kasih! |
| fr | Votre message a été envoyé. Merci ! |
| es | Tu mensaje ha sido enviado. ¡Gracias! |
| pl | Twoja wiadomość została wysłana. Dziękujemy! |

## Notes

- The backend already rate-limits `/support` by IP and IP+email and validates
  message length (5–5000) — the landing form needs no extra guard.
- Keep the form action server-rendered so it works without JS.
- `kind` for the landing form is `"feedback"` (vs the app's `"bug"`).
