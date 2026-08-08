# Prompt: SST Dotenv Files

This spec covers how environment (`dotenv`) files are treated inside the **`sst/`**
folder — the infrastructure-as-code. The same set of files is duplicated for the
mobile app; see [`mobile-app-env.md`](./mobile-app-env.md) for that side.

## The files

Every app keeps the same four dotenv files in `sst/`:

| File              | Purpose                                                    | Committed? |
| ----------------- | --------------------------------------------------------- | ---------- |
| `.env`            | Shared defaults / non-secret base values for all envs.    | Yes        |
| `.env.dev`        | Development stage values.                                 | Yes        |
| `.env.local`      | Per-developer machine overrides and secrets.              | **No**     |
| `.env.production` | Production stage values.                                  | Yes        |

> Note: filenames follow the standard `.env.<name>` convention. The shorthand
> `.env`, `.dev`, `.local`, `.production` maps to `.env`, `.env.dev`,
> `.env.local`, `.env.production`.

## Rules

1. **Scaffold all four.** When creating the `sst/` app, create `.env`,
   `.env.dev`, `.env.local`, and `.env.production`. These are duplicates of the
   four files in `expo/` — keep the two sets in sync in shape (same keys), even
   though the values differ.

2. **Never commit `.env.local`.** It holds machine-specific overrides and
   secrets. Add it to `.gitignore`. Commit the other three.

3. **Always provide `.env.local.example`.** Commit a `.env.local.example` listing
   every key required to deploy/run locally, with placeholder (never real)
   values.

4. **Precedence (lowest → highest).** `.env` → `.env.<stage>` → `.env.local`.
   `.env.local` always wins so a developer can override anything on their own
   machine.

5. **Stage ↔ file mapping.** The SST stage selects which file loads: `dev` stage
   reads `.env.dev`, `production` stage reads `.env.production`. `.env` is always
   loaded as the base underneath the stage-specific file.

6. **Prefer SST Secrets for real secrets.** Dotenv files are for non-sensitive
   config and local convenience. Production secrets (API keys, DB credentials)
   belong in SST's secret management (`sst secret set`) and are bound to
   resources via `Resource`/links — not committed in `.env.production`.

7. **Keys match across files.** Every key present in `.env` should appear in
   `.env.dev` and `.env.production` (even if empty) so it's obvious what each
   stage must define.

## Baseline layout

```
sst/
├── .env                  # shared defaults (committed)
├── .env.dev              # dev stage values (committed)
├── .env.local            # local secrets/overrides (gitignored)
├── .env.local.example    # template for .env.local (committed)
└── .env.production       # production stage values (committed)
```
