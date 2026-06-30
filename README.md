# Authentication — Technical Reference

What tokens/sessions are created during login, their lifetimes, and whether refresh tokens exist — for **Google** and **GitHub**. Based on the actual code in `backend/`, `web/`, and `mobile/`.

---

## 1. Two-token model (read this first)

Every login produces **two different tokens** with different jobs:

1. **Provider token** — issued by Google / GitHub. Used **once**, only during login, to prove identity and read the profile. The backend then **discards** it (never stored).
2. **App session token** — our **own JWT**, signed by the backend. This is what the app holds onto and sends on every later request. It is identical for both Google and GitHub logins.

So the provider tokens differ per provider, but the *session* the user ends up with is always our app JWT.

---

## 2. App session token (common to Google AND GitHub)

This is the answer to "what token / session is created after authentication."

- **Type:** JSON Web Token (JWT) — **not** a server-side session ID.
- **Signing:** `HS256` (HMAC + shared secret `JWT_SECRET`). Code: `jwt.sign(...)` in [backend/src/auth.js](backend/src/auth.js#L45-L57).
- **Claims inside it:**
  - `sub` — stable user key, format `"<provider>:<providerId>"` (e.g. `google:1234`, `github:567`)
  - `provider`, `email`, `name`, `picture`
  - `iat` (issued-at) and `exp` (expiry) — added automatically.
- **Lifetime:** `JWT_EXPIRES_IN`, default **`1h` (1 hour)**. See [backend/src/config.js](backend/src/config.js#L19).
- **Refresh token:** **None.** The backend issues no refresh token. When the 1-hour JWT expires, the user must sign in again.
- **Stateless / no session store:** the backend keeps **no session ID** and stores nothing to validate the token — it just re-verifies the signature on each request ([backend/src/auth.js](backend/src/auth.js#L60-L74)). The in-memory `users` Map only holds profile data, not sessions.
- **Where it lives (web):** browser `localStorage`, key `app_token` ([web/src/api.js](web/src/api.js#L3-L9)).
- **Where it lives (mobile):** in app memory only for GitHub/Facebook; the Google path on mobile builds the profile locally and does not mint an app JWT (see §3).
- **How it is sent:** HTTP header `Authorization: Bearer <jwt>` on protected calls like `GET /api/me`.

---

## 3. Google sign-in — background flow

**Endpoint:** `POST /auth/google` ([backend/src/index.js](backend/src/index.js#L36-L54))

Google offers two inputs; the backend accepts either:

### Path A — ID token (basic login, default on web)
- Web (`@react-oauth/google`) returns a **Google ID token** = a signed JWT (the `credential`).
- Backend verifies it with `google-auth-library` ([backend/src/auth.js](backend/src/auth.js#L14-L34)): checks Google's signature (against Google's public keys) **and** that the audience matches our client IDs.
- **Google ID token lifetime:** ~**1 hour** (set by Google).
- Used once for verification, then discarded.

### Path B — Access token (enrich / rich profile)
- An OAuth **access token** (opaque bearer string) is sent instead; the backend calls Google `userinfo` + **People API** for extra fields ([backend/src/people.js](backend/src/people.js)).
- **Google access token lifetime:** ~**1 hour** (set by Google).
- **Refresh token:** **not issued to us** — mobile is configured with `offlineAccess: false` ([mobile/App.js](mobile/App.js#L25-L28)), so Google returns no server-side refresh token. The backend never stores the access token either.
- On mobile, the native Google SDK can silently re-issue a fresh access token via `GoogleSignin.getTokens()` while the device's Google session is alive ([mobile/App.js](mobile/App.js#L108-L123)) — this is the SDK's own session, not our app's.

### Result
- Backend builds a normalized profile, then issues **our app JWT (1h, no refresh)** — see §2.

---

## 4. GitHub sign-in — background flow

**Endpoint:** `POST /auth/github` ([backend/src/index.js](backend/src/index.js#L60-L72)). GitHub has **no ID token**, so it uses the OAuth **authorization-code** flow.

1. **Authorization code** — client redirects the user to GitHub, gets back a short-lived `code`.
   - **Code lifetime:** ~**10 minutes**, **single use** (GitHub policy).
   - A random **`state`** value guards against CSRF: stored in `sessionStorage` on web and checked on return ([web/src/oauth.js](web/src/oauth.js#L34-L67), [web/src/pages/AuthCallback.jsx](web/src/pages/AuthCallback.jsx#L33-L40)).
   - **No PKCE** — the backend does a secret-based exchange instead (`usePKCE: false`, [mobile/socialAuth.js](mobile/socialAuth.js#L57-L63)).
2. **Code → access token (server-side):** the client POSTs `{ code, redirectUri }` to the backend; the backend exchanges it together with the **client secret** (which never leaves the server) for a **GitHub access token** ([backend/src/providers/github.js](backend/src/providers/github.js#L45-L66)).
   - **GitHub access token lifetime:** **does not expire by default** for OAuth Apps (no expiry unless the "expiring tokens" option is enabled).
   - **Refresh token:** **none** by default (refresh tokens only exist when expiring tokens are turned on).
   - The token is used immediately to read the profile (`/user`, `/user/emails`) and is **not stored** — discarded after the request.
3. **Result:** backend issues **our app JWT (1h, no refresh)** — same token as Google, see §2.

---

## 5. After login — using the session

- Protected route: `GET /api/me`, guarded by `requireAuth` ([backend/src/auth.js](backend/src/auth.js#L60-L74)).
- The middleware re-verifies the JWT signature + expiry on every call. Invalid/expired → **HTTP 401**.
- **Logout** = simply delete the stored app token client-side ([web/src/auth.jsx](web/src/auth.jsx#L49-L52)). There is nothing to revoke server-side because the session is stateless.

---

## 6. Summary table

| Item | Google | GitHub |
|---|---|---|
| Provider credential received | ID token (JWT) **or** access token | Authorization `code` |
| Code/token expiry (provider) | ID & access token ~1 hour | Code ~10 min; access token does **not** expire (default) |
| Provider refresh token | No (`offlineAccess: false`) | No (default) |
| Provider token stored by backend? | No | No |
| **App session token issued** | **JWT, HS256, 1 hour** | **JWT, HS256, 1 hour** |
| App refresh token | **None** | **None** |
| Server-side session ID? | No — stateless JWT | No — stateless JWT |
| On expiry | Re-login required | Re-login required |

---

## 7. Notes / POC limitations

- `JWT_SECRET` defaults to an insecure dev value — set a real one in production ([backend/src/config.js](backend/src/config.js#L58-L60)).
- User profiles live in an **in-memory Map** ([backend/src/index.js](backend/src/index.js#L16)); they vanish on restart. Use a real database in production.
- No refresh-token rotation: a 1-hour expiry means the user re-authenticates hourly. For longer sessions, add a refresh-token mechanism or increase `JWT_EXPIRES_IN`.

