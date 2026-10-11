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

### Going further (one line each)
- **Remember login across reloads:** pass `storage: new SessionStorageSessionStore('open4id.session.<appId>')` (see Sessions).
- **Full-page redirect instead of dialog:** `await auth.signInWithRedirect()`, then `await auth.handleCallback()` on the callback page (see Redirect sign-in).
- **Popup blocked:** catch `client.auth.popup_blocked` → `signInWithRedirect()` (see Popup-blocked fallback).

No bundler? Import from CDN instead of installing:
`https://unpkg.com/@open4id/sdk@0.2.0/dist/index.js`.
`@open4id/sdk/ui` needs a browser DOM; the root entry also runs in Node 18+.

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
except through its accessors.

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
and clear displayed state. Listen for it next to `open4id-signed-in`:

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
Native passwords require at least 12 characters; registration answers with a
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

### Password reset

```ts
await auth.requestPasswordReset('player@example.com'); // generic response
await auth.resetPassword('pat_...', 'a-brand-new-password'); // single-use 30-minute token
```

The request always answers success-shaped so responses never reveal
registration state. Resetting consumes the token, sets the bcrypt password
(minimum 12 characters), and revokes all sessions for the player; an
unknown or expired token throws `SdkAuthError('client.auth.reset_invalid')`.

The verification email links to the game origin's `/verify-email?token=`
page (the Angular example implements it mirroring the Publisher activate
page); the server-hosted `/api/client/verify_email` page remains as fallback.

### Redirect sign-in (full-page navigation)

`signInWithRedirect` builds the authorization URL and navigates the game
page to it via `location.assign` (it returns the URL it navigated to).
On the callback page — the exact registered `redirectUri` — call
`handleCallback()` to validate state and exchange the code for a session:

```ts
await auth.signInWithRedirect(); // leaves the game page
// ... on the callback page:
const session = await auth.handleCallback(); // defaults to the current page URL
```

Outside a browser (no `location.assign`) it throws
`SdkValidationError('client.auth.redirect_unavailable')` instead of
navigating.

### Popup sign-in (hosted page, game stays open)

`signInWithPopup` opens the same hosted authorize page in a popup window so
the game page stays open. The registered redirect page must call
`handlePopupCallback()` when it runs inside the popup: it forwards
`{code, state}` to the opener via `postMessage` and closes itself. The opener
validates the message origin against the redirect-URI origin, the message
source against the opened popup, and the state against the stored PKCE
transaction before exchanging the code. A blocked popup throws
`client.auth.popup_blocked`, a user-closed popup throws
`client.auth.popup_closed`, and an expired wait throws
`client.auth.popup_timeout`.

```ts
const session = await auth.signInWithPopup(); // resolves after the popup callback
```

```ts
// Inside the redirect page (for example /open4id/callback):
import { handlePopupCallback } from '@open4id/sdk';

if (window.opener && !window.opener.closed) {
  handlePopupCallback(); // forwards code/state and closes the popup
} else {
  await auth.handleCallback(); // full-page redirect exchange
}
```

No extra redirect URI registration is needed: the popup reuses the exact
registered redirect URI, and the redirect page branches on `window.opener`.

#### Popup-blocked fallback (explicit, host-owned)

The SDK opens the blank popup synchronously in the click call stack, but
strict blockers can still refuse it. The SDK never falls back to a
full-page redirect on its own: navigation wipes game state, and a
user-closed or denied popup must not yank the player to another page.
Fall back explicitly — and only on `popup_blocked`, which needs a working
callback route to land on:

```ts
try {
  await auth.signInWithPopup(); // or signInWithGooglePopup / GitHub / Apple
} catch (error) {
  if (error instanceof Error && error.message === 'client.auth.popup_blocked') {
    await auth.signInWithRedirect();
  } else {
    throw error;
  }
}
```

Host page checklist: serve over HTTPS, do not set
`Cross-Origin-Opener-Policy: same-origin` (it severs `window.opener`
from the callback page), register every callback/return URI (GitHub
OAuth App, Apple Services ID, Google authorized JavaScript origins),
and expect ad-blocked provider scripts as `system.service.down`, which
is another case for the same redirect fallback.

### Google popup sign-in (no navigation)

`signInWithGooglePopup` matches the Publisher Portal flow: the SDK lazily
loads Google Identity Services in the host page, opens the Google account
chooser in a popup with the application's public client ID, and posts the
short-lived Google code to `POST /client/player_google_authorize`. The
server exchanges it with the Vault-backed secret, resolves the player
(existing/linked/created, never merged; unverified provider email fails
closed), and answers the same single-use PKCE-bound code as usual, which
the SDK exchanges at `/token`. Nothing navigates, so the in-memory PKCE
transaction survives; the secret never leaves the server.

```ts
const session = await auth.signInWithGooglePopup();
```

A disabled provider throws
`SdkValidationError('client.auth.provider_unavailable')` before any popup.
A dismissed popup throws `SdkAuthError('client.auth.popup_closed')`, a
denial without a code throws `SdkAuthError('client.auth.access_denied')`,
and an expired wait throws `SdkAuthError('client.auth.popup_timeout')`.
When the Google script cannot load (for example an ad-blocker, or outside
a browser), it throws `SdkUnavailableError('system.service.down')`, which
the `<open4id-auth>` dialog reports as a sign-in failure. The live
round-trip needs the game origin registered as an authorized
JavaScript origin in the Google Cloud Console; it first passed on
2026-10-07 (GIS popup → game code → token pair → userinfo).

### GitHub popup sign-in

`signInWithGitHubPopup` matches the Publisher Portal flow: the SDK opens a
popup at `github.com/login/oauth/authorize` with the application's public
client ID and a game callback URI, the callback page
(`handleGitHubPopupCallback`) forwards the short-lived GitHub code to the
opener via `postMessage`, and the SDK posts it to
`POST /client/player_github_authorize` with the exact callback URI. The
server exchanges it with the Vault-backed secret, resolves the primary
verified email and the player (existing/linked/created, never merged;
missing or unverified email fails closed), and answers the same
single-use PKCE-bound code as usual, which the SDK exchanges at `/token`.
Nothing navigates in the host page, so the in-memory PKCE transaction
survives; the secret never leaves the server.

```ts
const session = await auth.signInWithGitHubPopup();
// const session = await auth.signInWithGitHubPopup({ callbackUri: 'http://localhost:5201/gh-callback' });
```

The callback URI defaults to `<redirectUri origin>/github-callback` and
must be registered in the GitHub OAuth App settings; only its origin must
be an allowed Web origin. Hosts that cannot serve an extensionless
callback path can point the dialog at another page with the
`github-callback-uri` attribute (or `setGitHubCallbackUri()`), which the
GitHub button forwards to `signInWithGitHubPopup({ callbackUri })`; that
exact page must be the registered one. A disabled provider throws
`SdkValidationError('client.auth.provider_unavailable')` before any popup.
A blocked popup throws `SdkAuthError('client.auth.popup_blocked')`, a
dismissed popup throws `SdkAuthError('client.auth.popup_closed')`, a
denial without a code throws `SdkAuthError('client.auth.access_denied')`,
and an expired wait throws `SdkAuthError('client.auth.popup_timeout')`.
The live round-trip needs the game callback URI registered in the GitHub
OAuth App plus a human click; first passed on 2026-10-07 (Product Owner
browser pass on the Angular example: dialog button → github.com popup →
game callback → token pair → session).

### Apple popup sign-in

`signInWithApplePopup` matches the Publisher Portal flow: the SDK lazily
loads AppleID JS in the host page, opens the Apple popup with the
application's public Services ID, and posts the short-lived Apple
authorization code plus the exact Return URL to
`POST /client/player_apple_authorize`. The server exchanges the code with
the Vault-backed private key (per-application client secret naming the
exact Return URL), verifies the returned identity token (signature,
issuer, audience, expiry, nonce binding), resolves the
player (existing/linked/created, never merged; missing or unverified email
fails closed), and answers the same single-use PKCE-bound code as usual,
which the SDK exchanges at `/token`. Nothing navigates, so the in-memory
PKCE transaction survives; the private key never leaves the server. The
same code-posting shape is ready for future native mobile callers.

```ts
const session = await auth.signInWithApplePopup();
// const session = await auth.signInWithApplePopup({ returnUri: 'https://game.example.com/apple-callback' });
```

The Return URL defaults to the redirect-URI origin with a trailing slash
(for example `https://open4id.com/`) and must be registered exactly —
including the trailing slash — in the Apple Services ID configuration
under Return URLs; otherwise Apple answers `invalid_request` / `Invalid
web redirect url`. Hosts that need another page can
point the dialog at it with the `apple-return-uri` attribute (or
`setAppleReturnUri()`), which the Apple button forwards to
`signInWithApplePopup({ returnUri })`; that exact page must be the
registered one and is named in the server-side exchange. A disabled provider throws
`SdkValidationError('client.auth.provider_unavailable')` before any popup.
A dismissed popup throws `SdkAuthError('client.auth.popup_closed')`, a
denial without a code throws
`SdkAuthError('client.auth.access_denied')`, a state mismatch throws
`SdkAuthError('client.auth.invalid_state')`, and an expired wait throws
`SdkAuthError('client.auth.popup_timeout')`. When the Apple script cannot
load (for example a content blocker, or outside a browser), it throws
`SdkUnavailableError('system.service.down')`. The live round-trip needs
the game Return URL registered in the Apple Services ID plus a human click;
it remains unverified, so no browser pass is claimed.

### Callback URI helpers

`deriveGitHubCallbackUri`, `deriveAppleReturnUri`, and
`deriveWebOrigin` expose the exact same rules the popup flows enforce,
for Publisher dashboards that preview what to register at each
provider console (no duplication, no drift):

```ts
import { deriveGitHubCallbackUri } from '@open4id/sdk';

deriveGitHubCallbackUri({ redirectUri, environment });
// → '<origin>/github-callback' (or the host override)
```

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

## Account deletion (Apple/Google compliance)

`deleteAccount` deletes the current player whatever its type (native,
guest, Google, GitHub, Apple). The access token proves ownership; the
server revokes every session and token family, erases the profile, all
linked identities, credentials, and action tokens, and answers 200. The
SDK clears its in-memory session (also when the server reports the
session is already invalid) and emits the signed-out state. Other
failures keep the session and throw. Provider-side grants are
additionally revoked best-effort using tokens captured at sign-in; any
miss never blocks the local delete.

```ts
if (window.confirm('Delete this player account permanently?')) {
  await auth.deleteAccount();
}
```

Calling without a session throws
`SdkAuthError('client.auth.not_authenticated')` before any fetch. The
`<open4id-auth>` delete-account mode wraps the same call as an explicit
confirmation step and emits `open4id-account-deleted`; both examples
open the dialog in that mode from their Delete account buttons.

## Errors

All failures throw typed `SdkError` subclasses keyed by stable string
codes. OAuth token errors map to typed SDK errors (`invalid_grant` →
`SdkAuthError`, `invalid_request`/`unsupported_grant_type` →
`SdkValidationError`, `temporarily_unavailable` → `SdkUnavailableError`).
The legacy `client.login` namespace still throws `sdk.not_implemented`;
`payments` and `entitlements` remain stubs.
