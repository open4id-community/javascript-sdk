# Changelog

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