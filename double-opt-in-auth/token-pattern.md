# The confirmation-token pattern

The reusable core. Every flow (support, reset, signup) is this with a different
parked payload and TTL. Reference impl: `/support` in `sst/src/index.ts`.

## Generate (on request)

```ts
import { randomBytes, createHash } from "node:crypto";

const id = randomBytes(16).toString("base64url");     // public row id (only when parking a row)
const token = randomBytes(32).toString("base64url");  // the secret — rides ONLY in the email URL
const confirmHash = createHash("sha256").update(token).digest("hex"); // the ONLY thing stored
```

## Store (hash + expiry, never the raw token)

Two storage shapes are in use:

**A. Parked row** (support/feedback, signup-pending) — a dedicated item that
self-expires via the table TTL (`expiresAt` in **epoch seconds**):

```ts
await putSupportRequest(id, {
  email, message, kind,
  confirmHash,
  createdAt: now,
  expiresAt: Math.floor(now / 1000) + 86400, // 24h — DynamoDB TTL attribute
});
```

**B. Hash on an existing record** (password reset) — stored on the user,
expiry in **epoch ms**, cleared on use:

```ts
await setResetToken(email, resetHash, now + 3600_000); // resetHash + resetExpiresAt
```

## Email the link (raw token in the URL only)

```ts
const origin = new URL(c.req.url).origin;
const link = `${origin}/support/confirm?id=${encodeURIComponent(id)}&token=${encodeURIComponent(token)}`;
await sendSupportConfirmEmail(email, link);
```

`email.ts` mailers keep a **local-dev fallback** — if not on Lambda, log the
link to the console instead of calling SES, so the flow is testable without a
verified identity:

```ts
const onLambda = () => !!process.env.AWS_LAMBDA_FUNCTION_NAME;
if (!onLambda()) { console.log(`[confirm -> ${to}] ${link}`); return; }
```

## Consume + verify (on click)

Single-use is enforced by **atomic fetch-and-delete** for parked rows, or by
**clearing the hash** for record-attached tokens.

```ts
// Parked row: delete returns the old item — one caller can ever win.
export async function consumeSupportRequest(id: string) {
  const res = await doc.send(new DeleteCommand({
    TableName: TABLE(), Key: { pk: `SUPPORT#${id}`, sk: "REQUEST" },
    ReturnValues: "ALL_OLD",
  }));
  return (res.Attributes as SupportRequest | undefined) ?? null;
}

// Verify: constant-time hash compare + TTL. Any failure → one generic message.
const rec = await consumeSupportRequest(id);
const presented = createHash("sha256").update(token).digest("hex");
const stored = rec?.confirmHash ?? "";
const sameLength = stored.length === presented.length;
const matches = sameLength && timingSafeEqual(Buffer.from(stored), Buffer.from(presented));
const live = !!(rec && Date.now() <= rec.expiresAt * 1000); // epoch-seconds row → ms
if (!rec || !matches || !live) return renderError("invalid or expired");
```

> `timingSafeEqual` throws on length mismatch — guard with the `sameLength`
> check first (both inputs are fixed-length hex here, but keep the guard).

## Checklist

- [ ] Token from a CSPRNG (`randomBytes`), ≥32 bytes, base64url.
- [ ] Persist **only** `sha256(token)`; raw token never written to DB or logs (prod).
- [ ] Expiry stored and enforced (epoch **seconds** for TTL rows, **ms** for record fields).
- [ ] Consume is atomic / clears the secret → link works exactly once.
- [ ] Constant-time compare; one generic failure message (no "expired" vs "wrong").
- [ ] Rate-limit the request endpoint **and** the confirm endpoint.
