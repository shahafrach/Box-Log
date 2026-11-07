# Auth port notes: Home -> Box Log

Phase 1 and 2 only. Home was read, not modified.

## 1. How Home does auth

**Stack.** React + `@supabase/supabase-js`, session in localStorage (`persistSession`, `autoRefreshToken`, own `storageKey`). The session lasts until logout. Refresh-token rotation is on (`config.toml`, reuse interval 10s). No `[auth.sessions]` timebox or inactivity timeout is set.

**Signup.** Invite-only. Public signup is off in the project config.
- A guide of several screens ("how it works") shows before signup. It is skipped on later visits (`guideSeen`).
- The Edge Function `register` creates the user with the admin API: email_confirm true, so no confirmation email. It checks the invite token and rate-limits by IP and by token. It signs the user in, then creates a one-time recovery code and returns it once. If any step fails it deletes the half-made account.
- Password rule: at least 10 characters, with a client hint and a server check. The server also maps the `weak_password` and `email_exists` errors.

**Login.** `signInWithPassword`. After 3 failed attempts a "forgot password" shortcut appears. Fields use `autocomplete` (username, current-password, new-password, one-time-code) and `dir="ltr"` for email, password and codes.

**Password reset.** Two paths:
1. Email: `resetPasswordForEmail(email, {redirectTo: <origin>/reset-password})`. The page waits for the recovery session, then calls `updateUser({password})`. Expired links show an "expired" step.
2. No email: the `recover-password` Edge Function takes email + recovery code + new password. It is rate-limited, answers every failure with the same error, revokes all sessions, rotates the code, and does not touch TOTP.

**Logout.** `signOut()` on the server first. Local data is cleared only after that succeeds (`signOutSafely`), so an offline failure keeps the queued writes. Home has no "log out everywhere" button in the UI. After a password change it calls `signOut({scope:'others'})`. On the server, `revoke_user_sessions()` revokes all of a user's sessions after a code-based reset.

**TOTP 2FA** (`src/lib/mfa.ts`, `MfaSetup.tsx`, `AccountSecurity.tsx`).
- Enroll: `mfa.enroll({factorType:'totp'})` returns a QR image and a secret. The UI shows both, the QR and a LTR manual key. Abandoned unverified factors are removed first. Then `challengeAndVerify` with a 6-digit code.
- Login: after the password step `getAuthenticatorAssuranceLevel()` gives `nextLevel === 'aal2' && currentLevel !== 'aal2'`, which shows the code screen.
- **AAL2 enforcement is client-side only.** `AuthGate` in `account.tsx` holds an aal1 session at the challenge screen. No RLS policy or migration in Home checks `aal`. That is fine for Home's data, but it means an aal1 token can read the API directly.
- Recovery: 10 one-time recovery codes through the `mfa-recovery` Edge Function (`generate` and `redeem`). Redeeming a code removes the TOTP factor, then `refreshSession`. Tables `mfa_recovery_codes` and `mfa_recovery_attempts` are service_role only. Codes are HMAC-hashed and attempts are rate-limited.
- Unenroll: `mfa.unenroll`. A password re-check runs on a throwaway client (`persistSession:false`) so the real session is not disturbed.

**Supabase pieces.**
- Auth settings: password length, redirect URLs (`site_url`, `additional_redirect_urls`), `enable_confirmations = false`, `[auth.mfa.totp]` enroll and verify enabled, signup off.
- Edge Functions: `register`, `recover-password`, `mfa-recovery`, `new-recovery-code`. They share `_shared/crypto.ts` (origin allow-list, HMAC, `bump_attempts`).
- Secrets: `RECOVERY_CODE_HMAC_SECRET`, `ALLOWED_ORIGINS`.
- Migrations: `...005_mfa_recovery_codes`, `...006_registration` (`revoke_user_sessions`, `find_user_id_by_email`, invites).
- MFA needs Supabase Pro on the hosted service. This is stated in the config comment.

**UI.** All strings in `src/i18n/features/auth_flow.ts`: Hebrew, RTL, with a mapped error table (`invalid_credentials`, rate limit, `insufficient_aal`, `session_expired`, offline). `role="status"` is used for messages and password show/hide is supported. A strict CSP means no inline script.

**PWA.** No iOS/Android-specific auth code. Session persistence is in localStorage. The reset link opens in the browser, not the installed app, so the reset page must be self-contained. Home has an app-lock gate on top.

## 2. Box Log today (actual state)

`app.js` (1764 lines), `index.html`, `sw.js`: a static, no-build, offline-first PWA. Data lives in IndexedDB only. The code has **no Supabase, no auth, no network API calls, no anonymous sign-in, no Brevo, no backend, no package.json, no migrations folder, no tests**. Its only external request is Google Fonts. The service worker is cache-first, with the cache name `box-log-v1.2.0` kept in sync with `APP_VERSION`. The language is English (LTR). Design tokens (`--bg`, `--chalk`, `--brass`, etc.) are CSS variables in `index.html`. The footer credit exists.

**This does not match the brief.** The brief assumes the app already has anonymous sign-in, Brevo reset email and app-managed Supabase config. None of that exists in this repo.

## 3. Gap list

| Item | Verdict |
|---|---|
| Supabase project, client, `supabase/` folder, migrations | **Add** (all new; the cloud project does not exist) |
| Anonymous sign-in + upgrade (`updateUser({email,password})` keeps the user id) | **Add** (new, nothing to preserve in the cloud yet) |
| Signup + explainer screens | **Add**. Not invite-only: Box Log has no invite system |
| Login, 10-char password rule, error table | **Add** |
| Password reset via Brevo from the app | **Add** (new Edge Function plus a Brevo key) |
| TOTP enroll, verify, aal2 gate, unenroll | **Add**. Port the behavior. The aal2 check should also be put in RLS (Home skips this) |
| Recovery codes for lost authenticator | **Add**, simplified |
| Logout, logout on other devices | **Add** (`signOut` with scope `global` / `others`) |
| Persistent session | **Add** (localStorage) |
| Cloud sync of workout data (what the account would protect) | **Not asked. Open question.** Without it, auth guards nothing |
| Offline-first, IndexedDB, design tokens, English LTR, credit line | **Keep** |
| supabase-js dependency (via CDN or bundler), CSP, SW rules for the Supabase host | **Change** |

## 4. Blockers for Phase 3 and 4

- No Supabase project, URL or anon key is available to me, and there is no Brevo key. A cloud project, MFA (Pro plan), the Auth redirect URLs, Edge Function secrets and a verified Brevo sender all need the dashboard. That conflicts with the "zero dashboard steps" target.
- I cannot run Supabase locally here to test TOTP and reset end to end.
- Box Log has no build step. Adding supabase-js means choosing a CDN import or adding a bundler.

