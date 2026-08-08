# Auth pages under your own domain + per-env deep-link redirect

Two correctness fixes a deployed sign-in flow needs, both surfaced by running the
[e2e](e2e-login-tests.md) / [self-test](self-test-page.md) flows against real
stages: (1) the server-rendered auth pages (forgot/reset/confirm) must be served
under the **product domain**, not a raw function URL; (2) each environment's
**deep-link redirect scheme** must be distinct so dev and prod don't collide on
one device.

Reference impl: `sst/landing page/app/lib/auth-proxy.server.ts`, the
`forgot`/`reset` routes, `safeRedirect` in `…/routes/login.tsx`, and
`AUTH_REDIRECT_URI` in `iosapp/src/config.ts`.

## 1. Reverse-proxy the auth pages onto the product domain

The Auth function (Lambda Function URL) server-renders forgot/reset pages whose
forms POST to relative paths. Instead of emailing users a raw
`…lambda-url….aws/reset` link, add thin Remix routes on the landing domain that
**proxy** to the Auth function, so the whole flow stays on
`https://<product-domain>/reset`.

```ts
// auth-proxy.server.ts — used by the /forgot and /reset routes (GET + POST).
export async function proxyAuthPage(request: Request, path: string): Promise<Response> {
  const base = (process.env.AUTH_BASE_URL ?? "").replace(/\/$/, "");
  if (!base) return new Response("This page isn’t available right now.", { status: 503, … });

  const search = new URL(request.url).search;
  const headers: Record<string, string> = {
    // Forward the visitor's IP so backend rate-limiting keys per USER, not per
    // (single) proxy egress IP — otherwise one IP rate-limits everyone.
    "x-forwarded-for": request.headers.get("x-forwarded-for") ?? "",
  };
  const init: RequestInit = { method: request.method, headers, redirect: "manual" };
  if (request.method !== "GET" && request.method !== "HEAD") {
    headers["content-type"] = request.headers.get("content-type") ?? "application/x-www-form-urlencoded";
    init.body = await request.text();
  }

  const res = await fetch(base + path + search, init);
  // Pass redirects through verbatim (e.g. a "back to the app" deep link).
  if (res.status >= 300 && res.status < 400) {
    const location = res.headers.get("location");
    if (location) return new Response(null, { status: res.status, headers: { location } });
  }
  const body = await res.text();
  return new Response(body, { status: res.status, headers: { "content-type": res.headers.get("content-type") ?? "text/html; charset=utf-8" } });
}
```

The reset **email link now points at `<LANDING_BASE_URL>/reset`**, and the forms
loop back through the same proxy routes (same relative paths), so the user never
sees the function URL.

Rules:
- **Forward `x-forwarded-for`.** A proxy collapses every visitor to one egress
  IP; without forwarding, per-user
  [rate-limiting](../general-specs/security/SEC-002-rate-limiting-and-lockout.md)
  buckets the whole world together.
- **`redirect: "manual"` + pass redirects through.** Don't let `fetch` silently
  follow a 3xx; relay status + `location` so app deep-links survive the hop.
- **Fail soft when unconfigured** — missing `AUTH_BASE_URL` → a 503 page, not a
  crash.
- **Relative form actions.** The proxied page must post to the same relative
  paths so submissions re-enter the proxy, not the function URL.

## 2. Per-environment deep-link redirect scheme

When dev and prod apps are installed on the same device, a single shared custom
URL scheme (`priorize://auth`) routes the redirect to *whichever* app claimed it
— a coin toss. Make the scheme **per-env** by deriving it from the (per-env)
bundle id, and have the backend allow-list enumerate every accepted scheme.

```ts
// iosapp/src/config.ts — env-specific, because BUNDLE_ID differs per stage
//   prod: tech.sp33c.three://auth   dev: tech.sp33c.three.dev://auth
export const AUTH_REDIRECT_URI = `${BUNDLE_ID}://auth`;
```

```ts
// landing /routes/login.tsx — only ever bounce back into a KNOWN app scheme.
function safeRedirect(uri: string): string | null {
  const allowed = ["priorize://", "tech.sp33c.three://", "tech.sp33c.three.dev://"];
  return allowed.some((p) => uri.startsWith(p)) ? uri : null;
}
```

Rules:
- **Scheme = `<bundle-id>://`,** and the bundle id is already per-env (see
  [DEL-003 — per-environment app identity](
  ../general-specs/delivery/DEL-003-per-environment-app-identity.md)), so dev and
  prod redirects can coexist.
- **Allow-list, never reflect.** `safeRedirect` returns the URI only if it
  starts with a known scheme; anything else → `null`. An open redirect that
  bounces into an arbitrary scheme is a phishing/exfil vector — see
  [AUTH-006 — secure web→app hand-off](
  ../general-specs/authentication/AUTH-006-secure-web-to-app-handoff.md).
- **Keep legacy schemes in the list** during a rename so already-installed
  builds keep working; drop them once those builds age out.
- **Takes effect on app rebuild** — the redirect URI is baked into the native
  build, so changing it requires a new app build, not just a backend deploy.

## Checklist

- [ ] User-facing auth links use the product domain, never the raw function URL.
- [ ] Proxy forwards `x-forwarded-for`; rate-limiting keys per user.
- [ ] Proxy uses manual redirects and relays 3xx `location`.
- [ ] Proxy fails soft (503) when its upstream base URL is unset.
- [ ] Redirect scheme derived from the per-env bundle id.
- [ ] Backend `safeRedirect` allow-lists every accepted scheme; rejects the rest.
- [ ] Legacy schemes retained until old builds age out.
