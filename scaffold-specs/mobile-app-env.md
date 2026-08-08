# Prompt: Mobile App (Expo) Dotenv Files

This spec covers how environment (`dotenv`) files are treated inside the **`expo/`**
folder — the native mobile app. The same set of files is duplicated for SST; see
[`sst-env.md`](./sst-env.md) for that side.

## The files

Every app keeps the same four dotenv files in `expo/`:

| File              | Purpose                                                    | Committed? |
| ----------------- | --------------------------------------------------------- | ---------- |
| `.env`            | Shared defaults / non-secret base values for all envs.    | Yes        |
| `.env.dev`        | Development environment values.                           | Yes        |
| `.env.local`      | Per-developer machine overrides and secrets.              | **No**     |
| `.env.production` | Production environment values.                            | Yes        |

> Note: filenames follow the standard `.env.<name>` convention. The shorthand
> `.env`, `.dev`, `.local`, `.production` maps to `.env`, `.env.dev`,
> `.env.local`, `.env.production`.

## Rules

1. **Scaffold all four.** When creating the `expo/` app, create `.env`,
   `.env.dev`, `.env.local`, and `.env.production`. The same four files are
   duplicated under `sst/` — keep the two sets in sync in shape (same keys), even
   though the values differ.

2. **Never commit `.env.local`.** It holds machine-specific overrides and
   secrets. Add it to `.gitignore`. Commit the other three.

3. **Always provide `.env.local.example`.** Commit a `.env.local.example` listing
   every key a developer must set locally, with placeholder (never real) values.

4. **Precedence (lowest → highest).** `.env` → `.env.<environment>` →
   `.env.local`. `.env.local` always wins so a developer can override anything on
   their own machine.

5. **Expo exposure.** Only variables prefixed with `EXPO_PUBLIC_` are bundled into
   the client and readable at runtime via `process.env`. Anything **not** so
   prefixed must be treated as build-time only and kept out of the shipped app.
   Never put a real secret behind an `EXPO_PUBLIC_` key — it ships to the device.

6. **Keys match across files.** Every key present in `.env` should appear in
   `.env.dev` and `.env.production` (even if empty) so it's obvious what each
   environment must define.

## Baseline layout

```
expo/
├── .env                  # shared defaults (committed)
├── .env.dev              # development values (committed)
├── .env.local            # local secrets/overrides (gitignored)
├── .env.local.example    # template for .env.local (committed)
└── .env.production       # production values (committed)
```
