# Open4ID minimal web sample

The smallest possible `@open4id/sdk` integration: one HTML file, no
framework, no bundler, no build step. It exercises the headless client
directly (`createAuthClient`): bootstrap discovery, email
register/sign-in, guest sign-in plus guest-to-email linking, session
refresh, sign-out, and account deletion.

It needs the published SDK (`@open4id/sdk@0.3.0` via `esm.sh`, pinned in
`index.html`), so publish the SDK first — see
`docs/JAVASCRIPT_SDK_RELEASING.md`.

## Prerequisites

The form comes prefilled with the Angular example's development config
(same App ID, environment, public key, and production endpoint), served
from the same origin (`http://localhost:5200`) with the same redirect
URI (`.../open4id/callback`) — everything is already registered for the
dev app, so it works with no dashboard changes. Replace the values with
your own application to use it elsewhere.

1. An Open4ID application with a development environment, a public SDK
   key (`sdk_pk_...`), and email/password + guest login enabled
   (Publisher Portal → Applications → Authentication).
2. The origin you serve this page from registered as an **Allowed web
   origin** (General) — preconfigured for `http://localhost:5200`.
3. The exact redirect URI registered (General → Exact OAuth redirect
   URIs) — preconfigured as `http://localhost:5200/open4id/callback`.
   Nothing navigates to it in this sample — dialog-less flows stay
   in-page — but every issued code is bound to it, so the strings must
   match character for character.

## Run

```sh
cd examples/web-sdk-minimal
npm start
# open http://localhost:5200 and press “1. Bootstrap”
```

(Stop the Angular example first — it uses the same port.)

Click through: **1. Bootstrap** (public discovery, no secrets) →
**2a. Register** (sends the 24h verification email) or **2b. Sign in**
/ **2c. Continue as guest** → **3. Refresh** (`ensureSession`, rotates
only near expiry) → **4. Sign out** (revokes server-side, clears
memory). SDK failures render as stable error keys (`client.*`,
`system.*`), safe to branch on and to file as bug reports.

## Notes

- Sessions live in SDK instance memory only and vanish on reload; the
  form values (never secrets — there are none in the browser) persist
  in `sessionStorage` for convenience.
- No redirect/popup flow here by design: those need a callback page
  plus a surviving PKCE transaction (see the Angular and vanilla
  examples). This sample covers the dialog-less flows that always work.
- Never put `sdk_sk_...`, provider secrets, or tokens in the page,
  its URL, or its logs. The only credential here is the public
  `sdk_pk_...` key, which is designed to ship in client code.
