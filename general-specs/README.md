# General Specs — cross-cutting requirements for all 2026 apps

This folder holds **product-wide** specifications: feature and non-functional
requirements that apply to **every** app we build, not to any single one. App-
specific behaviour lives in each app's own `specs/` folder; anything that should
be true *everywhere* lives here.

If a requirement would have to be copy-pasted into three or more apps, it belongs
here.

> 📑 **[INDEX.md](INDEX.md)** — sparse list of every group and spec.

## How specs are organised

One folder per requirement **group**. Each spec is one Markdown file with a
stable ID (`GROUP-NNN`). IDs never get reused, even if a spec is retired.

| Group | Prefix | Folder |
|-------|--------|--------|
| Public pages & routes | `PAGE` | [`pages/`](pages/) |
| User interface & design | `UI` | [`design/`](design/) |
| Authentication | `AUTH` | [`authentication/`](authentication/) |
| Security | `SEC` | [`security/`](security/) |
| Privacy & data protection | `PRIV` | [`privacy/`](privacy/) |
| Internationalization | `I18N` | [`internationalization/`](internationalization/) |
| Accessibility | `A11Y` | [`accessibility/`](accessibility/) |
| Reliability & observability | `REL` | [`reliability/`](reliability/) |
| Delivery & infrastructure | `DEL` | [`delivery/`](delivery/) |
| Data & API | `DATA` | [`data-and-api/`](data-and-api/) |

## Spec format

Every spec uses [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) keywords —
**MUST / MUST NOT / SHOULD / SHOULD NOT / MAY** — so conformance is testable. Each
file has: status, applicability, the normative requirement, rationale, acceptance
criteria, and per-app implementation notes.

**Status values:** `Adopted` (in force) · `Proposed` (drafted, not ratified) ·
`Deprecated` (superseded — see the replacement).

## Apps in scope

These are the apps the specs apply to ("all apps" = this list).

| App | Stack | Auth model | Clients |
|-----|-------|-----------|---------|
| **OpenCycle** | SST (Ion) + Hono on Lambda, AWS Cognito, DynamoDB | Cognito (SRP) + Sign in with Apple | Web + Expo iOS |
| **OpenOutdoor** | SST + Hono on Lambda, AWS Cognito, DynamoDB | Cognito (SRP) + Sign in with Apple | Web + Expo iOS |
| **Priorize** (Three Things) | SST + Hono on Lambda, DynamoDB | Self-issued tokens (scrypt + HMAC) + Apple | Web + Expo iOS |
| **Emergency** (Guarding Angel) | SST + Remix on Lambda, DynamoDB | Device key + one-time web handoff | Remix web + Expo native |
| **WebhookNotification** | SST + Remix + Lambda, DynamoDB, SES | Self-issued tokens (scrypt + HMAC) | Web + Expo app |
| **ClickUp API** | Remix proxy | Server-side API token (folder-scoped) | Internal |
| **Shopping List** | SwiftData + CloudKit mirroring, SST tips API | None (iCloud account held by the OS) | iOS + Mac Catalyst |

> Auth mechanisms differ per app on purpose; the **requirements** in these specs
> are mechanism-agnostic. A spec says *what* must hold (e.g. "no tokens in URLs"),
> not *which* library provides it.

## Conformance

Each spec lists per-app status in its "Implementation notes". A spec marked
`Adopted` that an app does not yet meet is a **gap** to be tracked in that app's
backlog, not a reason to weaken the spec.

_Last updated: 2026-09-06._
