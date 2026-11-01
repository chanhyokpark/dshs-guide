# Troubleshooting

One section per symptom. Login and token errors are listed in the 오류 table of [id.dshs.app/docs](https://id.dshs.app/docs), School API errors in the 오류 section of [api.dshs.app/docs](https://api.dshs.app/docs).

## redirect_uri mismatch

**Symptom:** id.dshs.app shows an error page with `invalid_redirect_uri` instead of the login screen, or `/token` rejects the code with `invalid_grant`.

**Cause:** the `redirect_uri` is not exactly one of the URIs registered for your Client (scheme, host, port, path and trailing slash all count), or it differs between `/auth` and `/token`.

**Fix:** send the registered string unchanged at both `/auth` and `/token`; for a new address (another `localhost` port, a new domain) ask the admin through [](dev-client.md). See [the error table](https://id.dshs.app/docs): its closest row is `invalid_request`, which lists an unregistered `redirect_uri` although the error page shows `invalid_redirect_uri`, and `invalid_grant` covers the `/token` case.

## invalid_scope

**Symptom:** the browser comes back to your `redirect_uri` with `error=invalid_scope`.

**Cause:** `scope` contains a Scope that does not exist or that your Client is not allowed to request.

**Fix:** request only Scopes from the list the admin sent you, spelled as in [the Scope list](dev-school-api.md#scopes), or ask for more. See `invalid_scope` in [the error table](https://id.dshs.app/docs).

## invalid_target

**Symptom:** `/auth` or `/token` answers `invalid_target`.

**Cause:** `resource` is not the School API base URL of the selected User's School, or the login asked for no School API Scope at all (`openid profile` only).

**Fix:** send the base URL from the School API reference (`https://api.dshs.app` here); an app that only logs users in leaves `resource` out. See [](dev-login.md#token-kind) and `invalid_target` in [the error table](https://id.dshs.app/docs).

## 401 after 15 minutes

**Symptom:** School API calls start failing with `401` about 15 minutes after login.

**Cause:** the access token expired; it lives 15 minutes.

**Fix:** refresh and retry the same request once, as in [Refresh on 401](dev-login.md#refresh-on-401). See the 학교 API `401` row in [the error table](https://id.dshs.app/docs).

## invalid_grant on refresh

**Symptom:** the refresh request to `/token` answers `400 invalid_grant`.

**Cause:** the refresh token went unused for 30 days, the user revoked consent or logged out, the one-year login limit passed, or an old refresh token was sent again after the 5-minute grace period (every refresh issues a new one).

**Fix:** log the user in again, and from then on always store the newest refresh token and never send an old one again. See `invalid_grant` in [the error table](https://id.dshs.app/docs).

## insufficient_scope

**Symptom:** a School API call answers `403` with `message: "insufficient_scope"`.

**Cause:** the token lacks a leaf Scope the route lists under 필요한 Scope; the `scope` value of the `WWW-Authenticate` header names it.

**Fix:** add the Scope to `scope` at `/auth` (ask the admin first if your Client is not allowed it) and let the user log in and consent again. A `403` with `forbidden` instead means the user's role, or ownership of the item, does not allow the action (editing someone else's post, for example), and no Scope changes that. See `403` in [the School API errors](https://api.dshs.app/docs).

## Users come back with only an id

**Symptom:** other Users in a response are just `{ "id": "…" }`, and name or seat fields are missing.

**Cause:** the token has no `user.read`; without `user.detail.read` the detail fields (grade, class, number and so on) are missing too.

**Fix:** request `user.read`, and `user.detail.read` if you need details; Users made private for the viewer's role still hide details. See [](dev-school-api.md#scopes).

## 429 Too Many Requests

**Symptom:** School API calls answer `429` with `message: "rate_limited"`.

**Cause:** your Client sent more than 120 requests in 60 seconds, counted across all of its users.

**Fix:** wait the number of seconds in `Retry-After`, cache responses, and watch `X-RateLimit-Remaining`. See [](dev-school-api.md#rate-limit) and `429` in [the School API errors](https://api.dshs.app/docs).

## CORS error in the browser

**Symptom:** the browser console blocks a request to api.dshs.app or id.dshs.app with a CORS error.

**Cause:** api.dshs.app allows every origin but without credentials, so a request with cookies (`credentials: 'include'`) fails; the token endpoint of id.dshs.app answers browsers only for SPA·네이티브 앱 Clients and only from the origin of one of their registered redirect URIs.

**Fix:** call api.dshs.app with the Bearer header and no cookies; exchange codes on your server if you have a server app, and serve an SPA from the origin of a registered redirect URI. See [](dev-login.md).

## Consent revoked by the user

**Symptom:** refreshes fail with `invalid_grant`, and the next login shows the consent page again.

**Cause:** the user revoked your app at [id.dshs.app/consents](https://id.dshs.app/consents), which invalidates every refresh token of your Client for that user.

**Fix:** log the user in again if they come back, and delete the data you stored under that consent, as the [usage rules](developer.md#usage-rules) require. See `invalid_grant` in [the error table](https://id.dshs.app/docs).

## user_locked or user_deactivated

**Symptom:** the School API answers `401` with `user_locked` or `user_deactivated`, or `/token` answers `invalid_grant` with that `error_description`.

**Cause:** the school locked or deactivated the User.

**Fix:** stop retrying and tell the user to contact the school; logging in again does not help until the User is unlocked. See `invalid_grant` in [the error table](https://id.dshs.app/docs) and `401` in [the School API errors](https://api.dshs.app/docs).
