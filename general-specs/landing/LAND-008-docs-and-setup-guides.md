# LAND-008 — Documentation & setup-guide pages

- **Status:** Proposed
- **Group:** Landing page
- **Applies to:** Public product documentation (`/docs`) and setup guides for an
  integration (`/ai`, `/integrations/<name>`) on an app's public site.
- **Last updated:** 2026-10-04

## Requirement

### Reachability

1. Developer-facing apps **MUST** publish a product manual at `/docs`, public and
   signed-out, linked from the nav ([LAND-003](LAND-003-responsive-navigation.md))
   and listed in the sitemap. Each integration that needs more than one sentence
   to set up **SHOULD** have its own guide page, linked from the landing section
   that mentions it.
2. Each page **MUST** have a table of contents, or sections with stable `id`
   anchors, so a support answer or an error message can link
   `/docs#templates` directly. Section ids are public URLs and follow
   [LAND-001](LAND-001-landing-page-anatomy.md) §4.

### Versioning

3. If the manual differs between app releases, versions are selected with a
   query parameter (`/docs?v=1.2.0`). Version ids are stable once published. The
   newest version is the default when there is no parameter.
4. Every version **MUST** canonicalize to the bare path (`/docs`), so versions
   don't fragment the index ([SEO-003](../seo/SEO-003-metadata-and-social-cards.md) §3).
   The version picker is a GET form that works without JavaScript. An unknown
   `?v=` falls back to the default, not to a 404.

### Commands and URLs people paste

5. **Derived from the deployment, never typed.** Every endpoint, host, or
   one-line setup command a page tells people to copy (an MCP endpoint,
   `claude mcp add …`, an API base URL, a `curl` example host) **MUST** be built
   server-side from the deployment's own configuration (environment or linked
   resource). It is shared by every page that shows it. A typed host goes stale
   on the first domain change, and people paste it anyway.
6. Each copyable block **MUST** carry a **copy button** that:
   - is a real `<button>` with an accessible name ("Copy command");
   - copies exactly the shown text, with no prompt symbol and no trailing
     newline;
   - confirms visibly for about 1 s (for example, a check icon);
   - works without breaking the layout on phones (the code wraps or scrolls
     inside its card, never the page).
7. **Secrets are never pre-filled.** Examples show placeholders or masked values
   (`x-secret: ••••••••`). A page **MUST NOT** render a signed-in user's real
   secret, token or API key into a copyable example.
8. **A protocol endpoint wins its path.** If a protocol endpoint and a docs page
   compete for the same path (`/mcp`), the endpoint keeps the path, because
   clients validate the pasted URL without following redirects. The docs move
   elsewhere (`/ai`). The docs page **MUST NOT** redirect to or shadow the
   endpoint.

### Content

9. The manual **SHOULD** cover, in order: quickstart (zero to first result),
   authentication, message or data formats, each integration, limits per plan
   (taken from the plan catalogue, [LAND-005](LAND-005-pricing-section.md) §10),
   and troubleshooting.
10. A guide for a connector that acts on the user's behalf (an AI agent, an
    automation platform) **MUST** include a **"what it can't do"** safety
    section and a **"privacy and revoking access"** section, with a working
    revoke path.
11. Docs and guides **MAY** be English-only
    ([LAND-004](LAND-004-language-and-locale-persistence.md) §7). The privacy
    statements they depend on stay in the localized privacy notice.

## Rationale

Docs are where a developer product earns or loses its first integration. The
rules here come from two production lessons:

- **Pasted URLs (§5, §8).** A connector URL typed into a page went stale. A docs
  page at the connector's own path failed the client's no-redirect URL check
  ("measured, twice").
- **Versioning without index fragmentation (§3, §4).** Each release otherwise
  adds a near-duplicate page to the index.

## Acceptance criteria

- [ ] `/docs` returns 200 signed-out, is linked from the nav and is in the sitemap.
- [ ] Every section has a stable anchor, and `/docs#<id>` jumps to it.
- [ ] `/docs?v=<old>` renders that version. An unknown `v` renders the default. All versions emit `canonical=/docs`.
- [ ] The version picker works without JavaScript.
- [ ] Changing the deployment's public host changes every pasted command and URL on the site, with no code edit.
- [ ] Each copy button has an accessible name, copies exactly the shown command and confirms visibly.
- [ ] No page renders a real secret into an example.
- [ ] The protocol endpoint path is served by the endpoint itself, not by a page or a redirect.
- [ ] Agent and automation guides contain "what it can't do" and revoke sections, and the revoke path works.

## Implementation notes

- **WebhookPush** is the reference:
  - `routes/docs.tsx` has `DOC_VERSIONS` (`1.2.0` default, `1.1.4`), a `?v=`
    GET form and `seo({ path: "/docs" })` for every version. Its sections are
    quickstart, auth, TradingView, format, templates, monitor, ai and limits.
  - `routes/ai.tsx` is the MCP connector guide. Its sections are what, ask,
    connect, tools, safety and privacy.
  - `lib/mcp.server.ts` builds `mcpEndpoint()` and `claudeMcpAddCommand()` from
    `MCP_PUBLIC_URL`, which `sst.config.ts` sets from the stage's web URL. These
    are shared by `/ai` and the landing page's `#ai` section.
  - `components/CopyButton.tsx` provides the copy button.
  - `/mcp` is a CloudFront cache behaviour to the MCP function, so Remix never
    sees it (§8).
- **Gap (§5):** `docs.tsx` hard-codes `const INGEST =
  "https://d1yykmxdakarpv.cloudfront.net"` and uses it in the quickstart `curl`
  and the TradingView URL. That is the same failure `mcp.server.ts` was written
  to prevent.
- **Gap (§9):** the plan limits in `/docs#limits` are prose, not drawn from
  `plans.ts`.
