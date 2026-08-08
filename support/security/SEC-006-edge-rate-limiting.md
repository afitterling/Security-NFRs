# SEC-006 — Edge rate limiting & DDoS protection

- **Status:** Adopted
- **Group:** Security
- **Applies to:** Any public site served through a CDN with origin compute
  (SSR/Lambda) behind it — especially sites with cost-bearing POST endpoints
  (contact/support forms, email-send). Backs [SEC-002](SEC-002-rate-limiting-and-lockout.md) §4
  at the edge layer.
- **Last updated:** 2026-06-20

## Requirement

1. A public site fronted by a CDN **MUST** attach a WAF / Web ACL at the edge so
   abusive requests are rejected **at the CDN, before origin compute runs**. The
   origin function **MUST NOT** be invoked for blocked requests (no Lambda spend,
   no DB write, no email send for a request the edge drops).
2. A broad **per-IP** rate-based rule **MUST** cap general request volume as a
   backstop against L7 floods and cache-busting (random query strings that skip
   the cache and hit the origin).
3. State-changing / cost-bearing requests (POST to the contact/support form,
   anything that sends email or writes a record) **MUST** have a **separate,
   tighter** per-IP limit scoped to those requests, since one POST costs more
   than one GET (Lambda + DB + email + sender-reputation risk).
4. A CloudFront-scoped Web ACL **MUST** be provisioned in **us-east-1**
   regardless of the origin region, defined as **infrastructure-as-code**, and
   gated to the **production** stage.
5. Over-limit responses **MUST** be a uniform edge `403`/`429` with **no origin
   invocation** and **no information disclosure** (no hint about the form,
   account, or backend).
6. Each rule **MUST** emit CloudWatch metrics + sampled requests so floods are
   observable and limits can be tuned.
7. Network / volumetric (L3/L4) DDoS is covered by the CDN's always-on shield
   (e.g. AWS Shield Standard) at no extra config — but this **MUST NOT** be
   relied on for application-layer (L7) abuse, which requires rules 1–3.

## Rationale

SSR/loader requests and form POSTs bypass the CDN cache and invoke origin
compute. Without an edge limiter, every request costs Lambda + DynamoDB + SES,
which enables **denial-of-wallet** (flood uncached routes until the bill or
account concurrency limit blows) and **email-bombing** (script the double-opt-in
form to send confirmation mail to arbitrary addresses, damaging sender
reputation and risking an SES sending pause). Rejecting floods at the edge keeps
them off the origin entirely; a tighter POST limit throttles the email vector
without affecting normal browsing.

## Acceptance criteria

- [ ] Flooding any path past the per-IP limit returns a `403` from the CDN, and
      origin logs show **no invocation** for the blocked requests.
- [ ] Rapid POSTs to the contact form trip the stricter POST limit well before
      the general per-IP limit.
- [ ] The WAF Web ACL exists in us-east-1, is attached to the production
      distribution (`webAclId`), and is defined in IaC — not click-ops.
- [ ] CloudWatch shows per-rule blocked-request metrics.
- [ ] Non-production stages are unaffected (no WAF cost on dev).

## Implementation notes

- **PingTray (`sp33c-landing`):** `sst.config.ts` provisions an
  `aws.wafv2.WebAcl` (scope `CLOUDFRONT`, via a dedicated `us-east-1` provider)
  with two rate-based rules:
  - `RateLimitPosts` — 50 req / 5 min / IP, scoped down to `method = POST`
    (the contact-form / email-bomb vector).
  - `RateLimitAll` — 2000 req / 5 min / IP, broad backstop for L7 / cache-busting
    floods against the SSR Lambda.
  Attached to the CloudFront distribution via the Remix component's
  `transform.cdn.distribution.webAclId`, production-only. L3/L4 is handled
  automatically by AWS Shield Standard on CloudFront.
- **Cost:** an AWS WAF Web ACL is ~US$5/mo + ~$1/mo per rule + ~$0.60 per million
  evaluated requests — order ~US$7/mo for a low-traffic marketing site, far below
  the denial-of-wallet exposure it caps.
- **Tuning:** limits are per-IP; revisit if a shared-NAT user base trips them, and
  consider AWS Managed Rules (common rule set, IP reputation) if abuse persists.
