# Prompt: Standard App Scaffold

Whenever I start a new app, scaffold the following folder structure:

- **`expo/`** — the native mobile app, built with Expo (React Native).
- **`sst/`** — the infrastructure-as-code, defined with SST. This folder may also contain:
  - **`sst/marketing/`** *(optional)* — the marketing site, built as a Remix app, nested as a subfolder inside the SST folder so its infrastructure is managed together.

So the baseline layout is:

```
my-app/
├── expo/              # native app (Expo / React Native)
└── sst/               # infrastructure code (SST)
    └── marketing/     # marketing site (Remix) — optional subfolder
```

Always set up `expo/` and `sst/` by default. Add `sst/marketing/` only when the app needs a marketing site.
