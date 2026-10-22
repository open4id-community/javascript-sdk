# Changelog

# Changelog

## 0.4.0

- **Shared Portal callback defaults (GitHub/Facebook only)**: the popup
  callback defaults now live on the Portal instead of the game origin —
  GitHub `https://app.open4id.com/github-callback`, Facebook
  `https://app.open4id.com/facebook-callback` (mirroring the native SDKs'
  per-app Portal bridges; `DEFAULT_PORTAL_ORIGIN` is exported). Both are
  plain GET redirects the Portal bridge completes, so they work from any
  page origin; the SDK packs the opener
  origin into the provider state, and the shared Portal callback pages
  target it when posting `{code, state}` back — cross-origin popups
  complete without the game hosting a callback page. Game-hosted callback
  pages keep working (unpacked states fall back to the page origin), and
  explicit `callbackUri` / element-attribute overrides still win (use them
  for localhost in development — GitHub/Facebook accept loopback HTTP).
  `deriveGitHubCallbackUri` / `deriveFacebookCallbackUri` no longer read
  the `redirectUri` input (accepted for compatibility); the Publisher
  preview shows the shared URL plus one bridge URL per enabled native
  platform.
- **Apple popup Return URL stays origin-derived, now HTTPS-only**: Apple's
  popup answers with `response_mode=web_message`, posting the response to
  the redirect_uri origin — a hard same-origin rule that a shared Portal
  Return URL cannot satisfy — so `resolveAppleReturnUri` keeps deriving
  the Return URL from the page's own origin (the registered redirect URI
  origin), and the Publisher preview shows one Return URL per registered
  redirect URI (their origins) plus the native bridges. Apple also never
  accepts HTTP (no localhost exception), so an HTTP origin is rejected
  locally with the dedicated `client.auth.https_required` code — mapped
  by the `<open4id-auth>` element to an actionable hint — instead of
  Apple's cryptic `invalid_request` inside the popup; local testing
  needs an HTTPS page origin (`https://localhost` with a local
  certificate, a tunnel, or a deployed environment).
- **Facebook Sign-In (popup)**: new `signInWithFacebookPopup()` on the
  headless client and the `<open4id-auth>` Facebook button. The SDK opens
  a popup at the Facebook OAuth dialog (pinned Graph API `v26.0`,
  `scope=email`) bound to the game callback URI (default the shared Portal
  bridge `https://app.open4id.com/facebook-callback`, overridable via
  `FacebookPopupSignInOptions.callbackUri` or the element's
  `facebook-callback-uri` attribute / `setFacebookCallbackUri()`), then
  posts `facebook_code` + `facebook_redirect_uri` to
  `POST /api/client/player_facebook_authorize` exactly like the GitHub
  flow; the server exchanges the code with the Vault-backed App Secret
  and answers the same single-use PKCE-bound authorization code. Requires
  the Facebook provider enabled in bootstrap (`login.facebook`); denial
  cancels with the popup error family. New exports:
  `handleFacebookPopupCallback()` (callback page postMessage forward),
  `FACEBOOK_CALLBACK_TYPE`, `deriveFacebookCallbackUri()` (Publisher
  preview derivation), and `FacebookPopupSignInOptions`. The GitHub
  waiter and callback-page handler were refactored into shared
  provider-agnostic internals — GitHub's exported shapes are unchanged.
- Examples gained `/facebook-callback` pages (minimal, UI-element,
  vanilla, Angular) and GitHub-parallel buttons; CDN pins synced to `0.4.0`.
    
## 0.3.1

- Forgot-password email format check: `requestPasswordReset()` rejects
  malformed addresses locally with `SdkValidationError`
  (`field_errors.email = invalid`, same shape rule as Core API) without
  any network call; unknown addresses still get the generic success so
  responses never reveal registration state. New `invalid_email` dialog
  text (en/vi) rendered by `<open4id-auth>` for both local and server
  (`400 client.player.invalid_request`) rejections.
- `SDK_VERSION` header (`X-Open4ID-SDK-Version`) synced to the package
  version (was stuck at `0.2.0`); README CDN pin updated to `0.3.1`.

## 0.3.0

- Safari popup fix (additive): new `preloadGooglePopup()` /
  `preloadApplePopup()` exports. The `<open4id-auth>` element warms both
  provider scripts when its buttons appear, and `signInWithGooglePopup()`
  / `signInWithApplePopup()` now open the provider popup synchronously
  inside the click call stack when preloaded (Safari blocks popups
  opened after an `await`); non-preloaded hosts keep the lazy fallback.
  A popup the browser still refuses throws `client.auth.popup_blocked`,
  which the dialog renders as an allow-popups hint instead of a generic
  failure. Custom-UI hosts should call the preload when their own
  provider buttons render.
- Docs: GitHub popup flow (`callbackUri` / `github-callback-uri`
  override), native per-app Portal bridge URLs
  (`.../github-callback/<package>` /
  `.../apple-callback/<package>`) for mobile hosts.

## 0.2.0

- Publisher server callback (opt-in, additive): after every fresh sign-in
  the SDK POSTs a login receipt
  (`{token_type, access_token, expires_in, scope, user}` — never a refresh
  token, verifier, or code) to the `server_callback` URL from bootstrap,
  gated strictly on its `enabled` checkbox. New `notifyServer()` manual
  replay, new `onServerCallbackError` listener, new
  `server_callback_disabled` / `server_callback_failed` error keys.
  Refresh rotation never re-notifies; delivery failure never invalidates
  the session. Verify server-side with
  `POST /api/server/player_verify_token` before trusting a receipt.
- New default endpoint: omitting `endpoint` now targets production
  (`https://id.open4id.com/api`, exported as `DEFAULT_ENDPOINT`) instead
  of failing validation. Explicit endpoints are unchanged.
- Docs: publisher receipt flow, production default, no self-host guidance.

## 0.1.3

- New `getConfig()` on `Open4IDAuthClient`: redacted-config passthrough
  (additive, backward compatible), so game code no longer needs the
  low-level client to display configuration.
- `Open4IDClient` and the `Open4ID` singleton are now marked
  internal/low-level (bootstrap only); new integrations should use
  `createAuthClient()`.
- Docs: README restructured around the two surfaces (quick start, dialog
  vs headless concepts, headless flows, sessions, errors); storage
  defaults made explicit (`MemorySessionStore` default, opt-in session/
  local stores) with the `storage` config shown in the quick start.

## 0.1.2

- `signInWithPopup` opens the blank popup synchronously in the click
  call stack before building the authorization URL (same pattern as the
  GitHub flow), so strict popup blockers no longer refuse it; an
  unnavigable popup fails fast with `popup_blocked`.
- New Publisher-tooling exports: `deriveGitHubCallbackUri`,
  `deriveAppleReturnUri`, `deriveWebOrigin` (plus `CallbackUriDerivation`
  type) — the exact rules the popup flows enforce.
- Malformed redirect URIs in the derivation helpers now throw typed
  `invalid_request` instead of a raw `TypeError`.
- Docs: popup-blocked fallback recipe (explicit, host-owned), GitHub
  repo quick start, `prepare` self-build for git installs and fresh
  clones.

## 0.1.0 — First release