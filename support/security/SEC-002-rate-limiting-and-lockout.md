# SEC-002 — Rate limiting & lockout

- **Status:** Adopted
- **Group:** Security
- **Applies to:** Every internet-facing endpoint, especially auth, send-email,
  and any per-invocation-cost path.
- **Last updated:** 2026-06-20

## Requirement

1. Authentication endpoints (login, signup, verify, resend, forgot, reset,
   token-mint, exchange) **MUST** be rate-limited by source IP **and** by a
   targeted key (IP+email / account) so neither broad floods nor single-account
   brute force succeed.
2. Repeated failed logins for one account **MUST** trip a temporary **lockout**
   that survives IP rotation.
3. Limiters **MUST** fail **open** (allow) on backing-store error, so an infra
   blip can't lock everyone out — but **MUST** still cap normal operation.
4. Endpoints whose every call costs money (push/email send, ingest, third-party
   fan-out) **MUST** have an application-level quota/burst guard, and **SHOULD**
   be backed by edge throttling (WAF/CloudFront/API Gateway) in production.
5. Over-limit responses **MUST** be uniform (HTTP 429 + neutral message) and
   **MUST NOT** reveal whether the account exists.
6. Counters **MUST** auto-expire (TTL) so the limiter store self-cleans.
7. **Page-level throttling:** every internet-facing page/route (not just auth and
   cost-bearing endpoints) **MUST** be rate-limited per source IP at a global
   budget (e.g. 60 requests/minute/IP, tuned per surface) so a single client
   can't drive unbounded load. Static assets and health checks **MAY** be
   exempted.
8. **DoS protection:** production deployments **MUST** sit behind edge DoS/DDoS
   mitigation (WAF / CDN / API Gateway) that absorbs volumetric and L7 floods
   before they reach the application, and **MUST** enforce per-IP and global
   concurrency/request ceilings. The application-level limiters in (1)–(7) are
   the second layer of defense, not the first.
9. **Failsafe message:** when a request is throttled, locked out, or shed under
   load, the user **MUST** see a single neutral, human-readable failsafe message
   (e.g. *"You're sending requests too quickly — please wait a moment and try
   again."*). It **MUST NOT** leak limits, account existence, or internal state,
   and **SHOULD** be returned even when limiters fail open or the edge sheds the
   request.
10. **Block non-human traffic, challenge when unsure:** automated/bot traffic
    that is not an allow-listed agent (search crawlers, uptime/health checks,
    declared API clients) **MUST** be handled at the edge via bot management.
    Three-way decision:
    - **Verified human** or allow-listed agent → pass.
    - **Confidently non-human** and not allow-listed → block with the neutral
      failsafe message from (9).
    - **Unsure / ambiguous** → **MUST** present a human check (JS/CAPTCHA/
      managed challenge) rather than hard-block; pass on success, block on
      failure or timeout. Never silently drop ambiguous traffic.

    The allow-list **MUST** be explicit and reviewable.

## Rationale

Without throttling, credential stuffing, code brute-forcing, inbox flooding, and
cost-bleed attacks are trivial. Fail-open + TTL keeps the control from becoming
its own outage or storage leak. Page-level limits and edge DoS mitigation cap
total load before app code runs; bot management strips automated abuse, but
challenging ambiguous traffic instead of blocking it avoids locking out real
users behind imperfect bot signals. A single neutral failsafe message keeps the
control from leaking account or limit state.

## Acceptance criteria

- [ ] Hammering `/login` from one IP returns 429 after the limit.
- [ ] N failed logins for an account lock it briefly even across IPs.
- [ ] Cost-bearing endpoints reject past quota and (prod) drop floods at the edge.
- [ ] Limiter rows carry a TTL and disappear after the window.
- [ ] Any page/route exceeding the per-IP global budget returns 429.
- [ ] A volumetric/L7 flood is absorbed at the edge before reaching app code.
- [ ] Confident bot traffic is blocked; ambiguous traffic gets a human check, not a hard block.
- [ ] Every throttle/lockout/shed path renders the same neutral failsafe message.

## Implementation notes

- **OpenCycle / OpenOutdoor:** `authstore.ts` `allow()` (fixed-window, fail-open, TTL) on all auth routes + per-email `isLockedOut`/`recordLoginFailure`; new `OpenOutdoorAuth`/`OpenCycleAuth` Dynamo tables.
- **Emergency:** `ratelimit.server.ts` on web login + handoff token mint.
- **WebhookNotification:** `lib/ratelimit.ts` — per-webhook burst, per-user daily quota, auto-disable on sustained overage; auth-attempt throttle; edge WAF noted as the real flood defense.
- **Priorize:** `rate-limit.ts` on `/web/authorize` + `/apple`.
