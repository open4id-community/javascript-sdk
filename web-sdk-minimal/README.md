# Open4ID minimal web sample

The smallest possible `@open4id/sdk` integration: one HTML file, no
framework, no bundler, no build step. It exercises the headless client
directly (`createAuthClient`): bootstrap discovery, email
register/sign-in, guest sign-in plus guest-to-email linking, Google /
GitHub / Facebook / Apple popup sign-in, session refresh, sign-out,
account deletion, app-open analytics, and the signed-out deletion
request.

It consumes the published `@open4id/sdk@1.0.0` from `esm.sh` (pinned in
`index.html`), so it always tracks a released version: move the pin when a
newer release ships.

## Prerequisites

The form comes prefilled with the demo development application's config
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
4. For the provider buttons (SDK 0.4.0 defaults):
   - **Google** needs this page's origin registered as an authorized
     JavaScript origin in the Google Cloud Console.
   - **GitHub** and **Facebook** need nothing game-side by default: the
     popups complete through the shared Portal callback bridges
     (`https://app.open4id.com/github-callback` and `/facebook-callback`)
     — the opener origin travels packed in the provider state. The
     `github-callback.html` / `facebook-callback.html` pages shipped in
     this folder are optional overrides: pass
     `signInWithGitHubPopup({ callbackUri })` /
     `signInWithFacebookPopup({ callbackUri })` to route the popups
     through your own hosted pages instead (then register those URLs in
     the GitHub OAuth App / Facebook app settings).
   - **Apple** needs the page origin registered as the Return URL in the
     Apple Services ID settings, and it must be **HTTPS**: Apple never
     accepts HTTP (no localhost exception), so over plain
     `http://localhost:5200` the Apple button fails fast with
     `client.auth.https_required` (the other providers keep working).
     Use an HTTPS origin (local certificate, tunnel, or deployed host)
     to exercise it.

## Run

```sh
cd examples/web-sdk-minimal
npm start
# open http://localhost:5200 and press “1. Bootstrap”
```

(The other web sample uses port 5200 too — stop it first.)

Click through: **1. Bootstrap** (public discovery, no secrets) →
**2a. Register** (sends the 24h verification email) or **2b. Sign in**
/ **2c. Continue as guest** / **2e–2h. Provider popups** (Google GIS,
GitHub / Facebook via the shared Portal callback bridges, Apple; no
navigation) → **3. Refresh** (`ensureSession`, rotates only near
expiry) → **4. Sign out** (revokes server-side, clears memory).
**6. Reset device id** re-mints the installation id and **7. Email
deletion link** starts the signed-out deletion flow. SDK
failures render as stable error keys (`client.*`, `system.*`), safe to
branch on and to file as bug reports.

## Notes

- Sessions live in SDK instance memory only and vanish on reload; the
  form values (never secrets — there are none in the browser) persist
  in `sessionStorage` for convenience.
- App-open analytics is wired on the cold start: creating the client
  shows the locally minted device id and sends one fire-and-forget
  `appOpen()` ping (it never throws and never blocks the page). The id is
  attribution metadata, not a credential — the server keeps only a salted
  hash — and **6. Reset device id** calls `resetDeviceId()`.
- Signed-out deletion stops at the request: **7. Email deletion link**
  calls `requestAccountDeletion(email)`, which always answers the same
  generic `202` for a well-formed address. The emailed link points at
  this origin's `/delete-account` page, which a real game hosts itself
  (the SDK has no confirm method), so this sample does not implement that
  page.
- No redirect flow here by design: that needs a callback route plus a
  surviving PKCE transaction, which this single-file sample does not set
  up.
  The provider popups are the in-page exception, and since SDK 0.4.0
  the GitHub/Facebook ones need no callback page of your own (shared
  Portal bridges). The `github-callback.html` / `facebook-callback.html`
  forwarders still ship in this folder for the game-hosted override
  path. This sample covers the dialog-less flows that always work.
- Never put `sdk_sk_...`, provider secrets, or tokens in the page,
  its URL, or its logs. The only credential here is the public
  `sdk_pk_...` key, which is designed to ship in client code.
