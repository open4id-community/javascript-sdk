# Open4ID minimal login-UI sample

The same backend as `../web-sdk-minimal`, but the login UI is the SDK's
`<open4id-auth>` dialog element instead of headless calls: one HTML
file, no framework, no bundler, no build step. Sign-in, sign-up,
forgot-password, guest, link-email, delete-account, and the
Google / GitHub / Facebook / Apple popups come with the element; this
page only creates the headless client, bootstraps it, binds both to the
element, and reacts to `open4id-signed-in` / `open4id-signed-out` /
`open4id-account-deleted`. The client is reused while the config is
unchanged, so the dialog remembers the session: reopening it shows the
account view, and the Account shortcuts jump straight into link-email,
forgot-password (change password), delete-account, or logout.

It needs the published SDK (`@open4id/sdk@0.4.1` via `esm.sh`, pinned in
`index.html`), so publish the SDK first — see
`docs/JAVASCRIPT_SDK_RELEASING.md`.

## Prerequisites

Prefilled with the Angular example's development config and served from
the same origin (`http://localhost:5200`) with the same redirect URI
(`.../open4id/callback`) — everything is already registered for the dev
app, so it works with no dashboard changes. Replace the values with your
own application to use it elsewhere.

## Run

```sh
cd examples/web-sdk-minimal-ui
npm start
# open http://localhost:5200 and press “Open login dialog”
```

(Uses port 5200 like the other local samples — run one at a time.)

## Notes

- Sessions live in SDK instance memory only; the dialog's provider
  buttons need no extra code (Google GIS / GitHub / Facebook / Apple
  popups, Safari-safe via the element's script preload). Under SDK
  0.4.0 the GitHub/Facebook popups complete through the shared Portal
  callback bridges by default; Apple requires an HTTPS page origin
  (plain HTTP fails with `client.auth.https_required`).
- The `github-callback.html` / `facebook-callback.html` pages shipped
  here are optional overrides for the game-hosted callback path: point
  the element's `github-callback-uri` / `facebook-callback-uri`
  attributes at them (like the vanilla example does) and register the
  URLs in the GitHub OAuth App / Facebook app settings.
- Never put `sdk_sk_...`, provider secrets, or tokens in the page. The
  only credential here is the public `sdk_pk_...` key.
