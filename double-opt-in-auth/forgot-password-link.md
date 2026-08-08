# Surface the "Forgot password?" link

The reset flow (`/forgot` + `/reset`) is fully built on the Auth backend, but
the landing login form never links to it. Add a localized link.

## Where

`sst/landing page/app/routes/_index.tsx` — the inline login/signup form. The
loader already exposes `authBase` (the Auth Function URL). The reset pages are
backend HTML at `${authBase}/forgot`.

(If `login.tsx` renders its own form, add the same link there.)

## What to add

Only show it in **login** mode, under the password field:

```tsx
{!isSignup && authBase && (
  <a className="auth-forgot" href={`${authBase}/forgot`}>
    {t.forgotPassword}
  </a>
)}
```

`authBase` is already destructured from `useLoaderData` in `_index.tsx`. No new
loader data is needed.

## i18n key (`app/i18n.ts`, all 8 langs)

`forgotPassword`:

| lang | text |
|---|---|
| en | Forgot password? |
| de | Passwort vergessen? |
| zh | 忘记密码？ |
| ms | Lupa kata laluan? |
| id | Lupa kata sandi? |
| fr | Mot de passe oublié ? |
| es | ¿Olvidaste tu contraseña? |
| pl | Nie pamiętasz hasła? |

## Notes

- It's a plain link to a backend page (full navigation), not an inline form —
  the reset flow renders + handles its own pages. No CORS / no API call.
- Don't prefill or pass the typed email in the URL (avoid leaking it in
  history/referrer); the `/forgot` page collects it itself.
- The `/forgot` page is already anti-enumeration + rate-limited, so linking it
  publicly is safe.
