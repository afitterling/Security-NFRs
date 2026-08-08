# Double opt-in signup (activation)

**Goal:** on signup, don't log the user in. Create the account *unconfirmed*,
email a confirmation link, and only activate (and start a session) when they
click it. Login is refused until confirmed.

## Current behaviour (to change)

`POST /web/authorize` (`sst/src/index.ts`) in `mode === "signup"`:
1. validates, rate-limits, anti-enumerates,
2. `await createUser({ email, salt, hash, createdAt: now })`,
3. **immediately issues a session token** (PKCE code or bearer).

`UserProfile` (`sst/src/db.ts`) has **no** confirmed/verified field, and login
has **no** confirmed check.

## Changes

### 1. Data model — `UserProfile` (`db.ts`)

```ts
export type UserProfile = {
  // …existing…
  /** Set when the user confirms their email via the signup link. Absent ⇒ unconfirmed. */
  confirmedAt?: number;            // epoch ms
  /** sha256 of the signup confirmation token; cleared once confirmed. */
  confirmHash?: string;
  confirmExpiresAt?: number;       // epoch ms
};
```

Add helpers mirroring the reset ones:
`setConfirmToken(email, hash, expiresAt)` and
`confirmUser(email)` (sets `confirmedAt`, clears `confirmHash`/`confirmExpiresAt`).

### 2. Signup — park unconfirmed, email link, DO NOT issue a session

```ts
if (mode === "signup") {
  const pwErr = passwordError(password); if (pwErr) return c.json({ error: pwErr }, 400);
  const existing = await getUser(email);
  if (existing) {
    // Existing UNCONFIRMED account → re-send link (idempotent), same neutral reply.
    if (!existing.confirmedAt) { /* regenerate token + sendSignupConfirmEmail */ }
    decoyVerify(password);
    return c.json({ error: "Couldn't create that account. If you already have one, log in instead." }, 400);
  }
  const { salt, hash } = hashPassword(password);
  await createUser({ email, salt, hash, createdAt: now }); // confirmedAt absent ⇒ pending

  const token = randomBytes(32).toString("base64url");
  const confirmHash = createHash("sha256").update(token).digest("hex");
  await setConfirmToken(email, confirmHash, now + 86400_000); // 24h
  const origin = new URL(c.req.url).origin;
  await sendSignupConfirmEmail(email, `${origin}/confirm?email=${encodeURIComponent(email)}&token=${encodeURIComponent(token)}`);

  // No session. Tell the client to show "check your email".
  return c.json({ pending: true, message: "Check your email to confirm your account." });
}
```

`sendSignupConfirmEmail(to, link)` — add to `email.ts`, mirror
`sendSupportConfirmEmail` (subject "Confirm your threethings account", keep the
local-dev console fallback).

### 3. Confirm endpoint

```ts
app.get("/confirm", async (c) => {
  const email = String(c.req.query("email") ?? "").trim().toLowerCase();
  const token = String(c.req.query("token") ?? "");
  if (!email || !token) return c.html(renderConfirmPage({ error: "invalid" }));
  if (!(await allow(`confirm:ip:${clientIp(c)}`, { limit: 30, windowSec: 600 }, Date.now())))
    return c.html(renderConfirmPage({ error: "rate" }));

  const user = await getUser(email);
  const presented = createHash("sha256").update(token).digest("hex");
  const stored = user?.confirmHash ?? "";
  const ok = stored.length === presented.length
    && timingSafeEqual(Buffer.from(stored), Buffer.from(presented))
    && !!(user?.confirmExpiresAt && Date.now() <= user.confirmExpiresAt);
  if (!user) return c.html(renderConfirmPage({ error: "invalid" }));
  if (user.confirmedAt) return c.html(renderConfirmPage({ ok: true })); // already done → idempotent
  if (!ok) return c.html(renderConfirmPage({ error: "invalid" }));

  await confirmUser(email); // sets confirmedAt, clears the hash → single-use
  return c.html(renderConfirmPage({ ok: true })); // "You're in — open the app / log in"
});
```

### 4. Login guard (`mode === "login"`)

After password verification succeeds, before issuing the token:

```ts
if (!user.confirmedAt && user.provider !== "apple") {
  return c.json({ error: "Confirm your email first — check your inbox.", needsConfirm: true }, 403);
}
```

Expose a resend path (re-`POST /web/authorize` signup of an unconfirmed email
re-sends, or a dedicated `POST /resend-confirm` with its own rate bucket).

## Edge cases

- **Legacy accounts** (created before this feature) have no `confirmedAt`.
  *Decision needed:* grandfather them — treat `createdAt < FEATURE_CUTOFF_MS` as
  confirmed in the login guard, so existing users aren't locked out.
- **Apple sign-in** accounts are inherently verified → skip the guard
  (`provider === "apple"`).
- **Re-signup of an unconfirmed email** → regenerate token + resend, never a
  hard "already exists" that enables enumeration.
- **Expired link** → generic invalid message + offer resend.
- **Sessions:** signup no longer returns a token; the PKCE/native path must also
  branch on `pending` and show a "confirm your email" screen instead of
  completing the deep-link.
