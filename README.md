# Open4ID Web SDK

[![npm version](https://img.shields.io/npm/v/@open4id/sdk)](https://www.npmjs.com/package/@open4id/sdk)
[![license](https://img.shields.io/npm/l/@open4id/sdk)](./LICENSE)

Framework-agnostic JavaScript SDK for Open4ID player authentication:
OAuth Authorization Code + PKCE, social provider popups (Google, GitHub,
Apple), guest sessions, and a Shadow-DOM `<open4id-auth>` element. No
runtime dependencies.

## Quick start: the dialog element in 5 steps

> Recommended path: a drop-in `<open4id-auth>` dialog that handles
> email/password, sign-up, guest, and social popups with no navigation
> on your side. The sections below document the pieces: the dialog in
> detail, the headless client for custom UI, and session handling.

### Step 0 — Publisher setup (dashboard, once)
Register the exact redirect URI (e.g. `https://game.example.com/open4id/callback`)
and the game origin in your Open4ID application. The Authentication page
shows the exact callback values to paste at each provider console
(Google / GitHub / Apple) — copy them character for character.

### Step 1 — Install

```sh
npm install @open4id/sdk
```

### Step 2 — Create the client and bootstrap once

```ts
import { createAuthClient, SessionStorageSessionStore } from '@open4id/sdk';

const auth = createAuthClient({
  appId: 'app_0123abcd',               // from your Open4ID application
  environment: 'production',           // development | sandbox | production
  publicSdkKey: 'sdk_pk_...',         // public key only — never the secret key
  endpoint: 'https://id.open4id.com/api', // optional — this production URL is the default; development uses its own URL
  redirectUri: 'https://game.example.com/open4id/callback', // exact registered URI
  // Opt-in: remember login across reloads (default is memory-only, lost on reload).
  // storage: new SessionStorageSessionStore('open4id.session.app_0123abcd'),
});

await auth.bootstrap({ platform: 'web' });
```

### Step 3 — Drop in the login dialog

```html
<dialog id="login-dialog">
  <open4id-auth id="login"></open4id-auth>
</dialog>
```

```ts
import { defineOpen4IDAuthElement } from '@open4id/sdk/ui';

defineOpen4IDAuthElement();
// `!`: the element exists (same markup above); showGameLobby/showLandingPage are yours.
const login = document.querySelector('open4id-auth')!;
const dialog = document.querySelector('dialog')!;
login.setBootstrap(auth.getBootstrap());
login.setAuthClient(auth);
login.addEventListener('open4id-signed-in', () => {
  dialog.close();
  showGameLobby(auth.getSession());
});
login.addEventListener('open4id-signed-out', () => showLandingPage());
login.addEventListener('open4id-account-deleted', () => showLandingPage());

dialog.showModal(); // open it from your Login button
```

Email/password, sign-up, guest, Google, GitHub, and Apple all render
inside — no provider code on your side.

### Step 4 — Use the session

```ts
auth.getSession();        // { accessToken, expiresAt, scope, userId } or null
auth.getCachedUser();     // { sub, email? } or null
await auth.ensureSession(); // no-op when still valid, rotates when near expiry
```

### Step 5 — Refresh, sign out, delete

```ts
await auth.ensureSession();  // call before important calls
await auth.signOut();        // revoke server-side, clear local state
await auth.deleteAccount();  // permanent; needs a signed-in session
```

### Notice:
- **Remember login across reloads:** pass `storage: new SessionStorageSessionStore('open4id.session.<appId>')` (see Sessions).

Direct using? Import from CDN instead of installing:
`https://unpkg.com/@open4id/sdk@0.3.1/dist/index.js`.

## Concepts: dialog element vs headless client

The SDK has two surfaces over the same protocol (OAuth Authorization Code
+ PKCE, same token endpoint, same session shape):

- **Dialog element** (`<open4id-auth>`, `@open4id/sdk/ui`): the
  recommended path used in the quick start. It renders every sign-in
  method and calls the headless client for you — no provider or token
  code on your side.
- **Headless client** (`createAuthClient()`): use it directly for a
  custom UI, a redirect-only flow, or a single popup call without the
  dialog.

`bootstrap()` always makes a request; `getBootstrap()` only returns the latest
in-memory successful snapshot. For browser calls, omit `origin` to use
`location.origin`. For Node or SSR, provide `origin` explicitly; the SDK sends
a matching `Origin` request header.

## Dialog element (`@open4id/sdk/ui`)

`@open4id/sdk/ui` provides a Shadow-DOM Player Auth layout bound to a
headless client. It renders sign-in, sign-up, forgot-password,
link-email, and delete-account modes
from safe bootstrap readiness, and every protocol operation runs in the
bound headless client: credentials and tokens never leave the client
except through its accessors. It shows only the login methods the
Publisher enabled for the environment (normal email/password, guest,
Google, GitHub, Apple) — toggled in the Publisher Portal under App
Settings → SDK & API keys → Login methods — and hides the rest
automatically after the next bootstrap.

```ts
import { defineOpen4IDAuthElement } from '@open4id/sdk/ui';

defineOpen4IDAuthElement();
const login = document.querySelector('open4id-auth');
login?.setBootstrap(auth.getBootstrap());
login?.setAuthClient(auth);
login?.addEventListener('open4id-signed-in', (event) => {
  // (event as CustomEvent).detail.provider: 'email_password' | 'guest' | 'google' | 'github' | 'apple'
  // read auth.getSession() / auth.getCachedUser() and close the dialog
});
```

The element is session-aware: while the bound client holds a session,
the entry modes (`sign-in`/`sign-up`) render the signed-in account view
(player identity, sign out, delete-account entry) instead of the login
form, and any session change re-renders through the client's
auth-state listener. A session past the client expiry skew renders the
same view with a refresh prompt (`ensureSession` in place) instead of
dropping back to login. Explicit task modes (`forgot-password`,
`link-email`, `delete-account`) always render their own view. Signing
out inside the element emits `open4id-signed-out` next to the other two
events.

Sign-in submits email/password through `signInWithPassword` and reports
wrong credentials or a pending email verification inline (with a working
resend-verification action). Sign-up submits through `register` and points
at the verification email. Forgot-password submits through
`requestPasswordReset` and points at the reset email (single-use 30-minute
link consumed on the game's reset page). The guest button continues as a
guest through `continueAsGuest`. The link-email mode (`mode="link-email"`
or `setMode('link-email')`) links the guest session to a real address
through `linkEmail`: taken addresses and non-guest sessions get inline
guidance instead of raw errors. Google brokering has a backend slice, so
the Google button signs in through `signInWithGooglePopup` (GIS popup, no
navigation); GitHub brokering has a backend slice too, so the GitHub button
signs in through `signInWithGitHubPopup` (popup at `github.com` plus a game
callback page); Apple brokering has a backend slice too, so the Apple
button signs in through `signInWithApplePopup` (AppleID popup, no
navigation). The component never parses,
validates, stores, or exposes player tokens.

The delete-account mode (`mode="delete-account"` or
`setMode('delete-account')`) is an explicit confirmation step for the
signed-in player: it calls headless `deleteAccount()` and emits
`open4id-account-deleted` on success so the host can close its dialog
and clear displayed state. The entry is shown only while the publisher's
delete-button toggle stays enabled (SDK keys page); a disabled toggle
hides it and the endpoint answers `403`. Listen for it next to `open4id-signed-in`:

```ts
login?.addEventListener('open4id-account-deleted', () => {
  // session is already cleared in the client; close the dialog
});
```

The dialog text is localized: `en` by default, `vi` built in. Set the
language with the `lang` attribute or `setLocale()`:

```html
<open4id-auth lang="vi"></open4id-auth>
```

```ts
dialog.setLocale('vi'); // 'vi-VN' also works; anything else falls back to English
```

The dialog brand names the application instead of a generic product
mark: it reads the bootstrap `displayName` (the application's configured
name, served by Core API bootstrap v2) and renders it with its initial
as the mark, including in the sign-up intro. Override it explicitly when
the game knows its own name without waiting for bootstrap:

```html
<open4id-auth app-name="My Game"></open4id-auth>
```

```ts
dialog.setAppName('My Game'); // setAppName(null) clears the override
```

Without an override or a configured name, the dialog keeps the generic
Open4ID brand.

Headless error keys (`client.auth.*`) are a stable contract and stay
untranslated; only the rendered dialog text is localized.

## Headless flows

Call the client directly when you own the UI instead of `<open4id-auth>`.
Every flow below ends in the same `AuthSession` (read it through
`getSession()` / `getCachedUser()`, keep it alive with `ensureSession()`
— see Sessions). Guest sign-in (`continueAsGuest`) and guest-to-email
linking (`linkEmail`) follow the same single-use code exchange; the
dialog wires them to its guest and link-email buttons.

### Email/password, registration, and verification

Prerequisites on the Application/environment: web enabled, an OAuth client ID
issued (Publisher Authentication save generates `oc_web_...`), the exact
redirect URI and web origin registered, and email/password login enabled.
Native passwords require at least 8 characters; registration answers with a
generic accepted response even for invalid input (anti-enumeration), so a
short password only surfaces later as a sign-in failure.

Registration sends a 24h verification email (same `send_mail` path as
Publisher activation). An unverified address may sign in while a verification
token is outstanding; afterwards sign-in fails until re-verified:

```ts
await auth.register('player@example.com', 'sufficient-long-password');
await auth.resendVerification('player@example.com'); // generic response
await auth.verifyEmail('pat_...'); // single-use token from email or verify page
```

`signInWithPassword` powers first-party game dialogs such as
`<open4id-auth>`: the credentials travel once to
`POST /client/player_authorize`, which answers the same single-use,
PKCE-bound authorization code the hosted form would issue, and the SDK
exchanges it at `/token` exactly as in the redirect/popup flows.

```ts
const session = await auth.signInWithPassword('player@example.com', 'sufficient-long-password');
```

Wrong credentials throw `SdkAuthError('client.player.invalid_credentials')`;
an address past its verification grace throws
`SdkAuthError('client.player.email_not_verified')` instead of a session.

### Social popups

```ts
const session = await auth.signInWithGooglePopup();
```

Safari only opens the Google chooser when the popup starts inside the
button-click call stack. The dialog element preloads the Google script
when its Google button appears, so the click path stays synchronous;
custom-UI hosts calling `signInWithGooglePopup()` directly should call
`preloadGooglePopup()` when their own Google button renders. A popup
the browser still refuses throws
`SdkAuthError('client.auth.popup_blocked')`.

```ts
const session = await auth.signInWithApplePopup();
```

The same Safari rule applies to the Apple popup: the dialog element
preloads the AppleID script with its Apple button, and custom-UI hosts
should call `preloadApplePopup()` when their own Apple button renders.

GitHub signs in through `signInWithGitHubPopup()` (popup at
`github.com` plus a game callback page that forwards the provider code
with `handleGitHubPopupCallback()`; pass `callbackUri` explicitly, or
set the element's `github-callback-uri` attribute, when the game cannot
serve the default `<origin>/github-callback` path). Native mobile apps
call back through the Portal first instead of a game page: one bridge
URL per application (`https://app.open4id.com/github-callback/<package>`
/ `.../apple-callback/<package>`), passed as the GitHub callback URI /
Apple Return URL — see the Android SDK README.

### Password reset

```ts
await auth.requestPasswordReset('player@example.com'); // generic 202 for any well-formed address; malformed throws SdkValidationError email=invalid
await auth.resetPassword('pat_...', 'a-brand-new-password'); // single-use 30-minute token
```

The request answers success-shaped for any well-formed address so responses never reveal
registration state. Resetting consumes the token, sets the bcrypt password
(minimum 8 characters), and revokes all sessions for the player; an
unknown or expired token throws `SdkAuthError('client.auth.reset_invalid')`.

The verification email links to the game origin's `/verify-email?token=`
page (the Angular example implements it mirroring the Publisher activate
page); the server-hosted `/api/client/verify_email` page remains as fallback.

## Sessions

### Refresh

Access tokens live 15 minutes; the SDK never refreshes by itself. Prefer
`ensureSession(minValidityMs)` (Keycloak-style `updateToken`): it returns
the session untouched when it outlives the window and rotates once via
`refresh()` otherwise (default window 30s, matching `isAuthenticated()`).
Call it before important calls, when an API call fails with an expired
token, or periodically while a long game session is open.
Refresh-token families live 30 days idle and 90 days absolute; every
refresh rotates to a new refresh token and the previous one dies
immediately.

```ts
await auth.ensureSession(30_000); // no-op when still valid, one rotation otherwise
```

Concurrent `refresh()` calls are single-flight: they share one rotation
instead of racing (a race would reuse a consumed token, which the server
treats as theft, revokes the whole family, and clears the local
session). Concurrent logins and refreshes are serialized to complete in
start order (deterministic last-starter-wins) instead of interleaving
exchanges and clobbering each other's session. A sign-out, account
deletion, or reset that lands while a refresh or login is in flight wins:
the late result is discarded instead of resurrecting the cleared
session. Only idempotent reads retry once after a network flap or
timeout; single-use exchanges (code, refresh, authorize, delete) never
retry, because a replay can consume-or-replay server-side. A rejected
refresh (`invalid_grant`, OAuth or envelope shape) always clears local
state, so treat it as signed-out and send the player back to sign-in
instead of retrying.

### Persistence (opt-in storage adapters)

The client is memory-only by default (`MemorySessionStore`) and forgets
the session on reload — it never touches `localStorage` unless you ask
it to. Pass a session store to survive reloads; every established/rotated
session is saved, and configuring the client picks it back up
(malformed or foreign-app blobs are dropped). A restored session with
an expired access token is still usable: `ensureSession()` rotates it
when the refresh token is alive.

| Store | Survives | Default? | Note |
|---|---|---|---|
| `MemorySessionStore` | Nothing (lost on reload) | Yes | Safest; re-login every load |
| `SessionStorageSessionStore` | Reloads within one tab | No | Recommended; safer on shared machines |
| `LocalStorageSessionStore` | Tabs and browser restarts | No | Most convenient, highest XSS exposure |

```ts
import { createAuthClient, SessionStorageSessionStore } from '@open4id/sdk';

const auth = createAuthClient({
  ...config,
  storage: new SessionStorageSessionStore(`open4id.session.${appId}`),
});
```

Tokens at rest are readable by page JavaScript, so XSS exposure is the
trade-off behind the table above: rotation plus the 90-day absolute family limit is the
mitigation, and the key should always include the Application ID so
games sharing an origin never read each other's sessions. A custom
`SessionStore` (`load`/`save`/`clear`, never throwing) works the same
way; sign-out, deletion, reset, and reuse-detection wipe the stored
copy like the memory one.

### Publisher server callback (optional login receipt)

Publishers with their own backend can tick the `server_callback` checkbox
in the Authentication configuration and save a URL. After every fresh
sign-in the SDK POSTs one login receipt there:

```json
{ "token_type": "Bearer", "access_token": "oa_...", "expires_in": 900,
  "scope": "openid profile email", "user": { "sub": "...", "email": "..." } }
```

Rules: URL presence alone never triggers it (`enabled` must be `true`);
refresh rotation never re-notifies; a delivery failure never invalidates
the session (subscribe with `onServerCallbackError`, or replay manually
with `await auth.notifyServer()`). The receipt never carries a refresh
token, verifier, or code. The Publisher backend verifies the access token
with `POST /api/server/player_verify_token` (server-to-server namespace,
`sdk_sk_...` auth alone — no public key needed, app-bound `{active, ...}`
answer) before trusting it, and owns CORS on its own URL. Publishers
without a backend ignore this entirely: the default flow is unchanged.
