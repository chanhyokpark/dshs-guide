# Log in with dshs.app

Log in with dshs.app is standard OpenID Connect (Authorization Code flow) served by `https://id.dshs.app`. This page gives you the choices and copy-paste code; [id.dshs.app/docs](https://id.dshs.app/docs) explains every parameter, token and error in full.

## Choose a Client type

- **Server app** (서버 앱, confidential Client): your server exchanges the code, authenticating with `client_secret`. PKCE is optional.
- **SPA or native app** (SPA·네이티브 앱, public Client): the browser or the device exchanges the code. There is no secret, so PKCE (S256) is required.

Pick server app whenever you have a backend; the Client 종류 table in [the reference](https://id.dshs.app/docs) lists the remaining differences, such as how often the consent page (동의 화면) appears.

## The flow in five steps

1. **Log in.** Send the user's browser to [`GET /auth`](https://id.dshs.app/docs#tag/로그인/GET/auth) with your `client_id`, `redirect_uri`, `scope` and `state`. The user signs in on id.dshs.app, picks one of their Users and consents to your Scopes.
2. **Exchange the code.** id.dshs.app sends the browser back to your `redirect_uri` with `code` (valid for 60 seconds, single use) and `state`. Post the code to [`POST /token`](https://id.dshs.app/docs#tag/토큰/POST/token) with your `client_secret` (server app) or `code_verifier` (SPA) and `resource=https://api.dshs.app` to get `access_token`, `id_token` and, if you asked for `offline_access`, `refresh_token`.
3. **Know who logged in.** The `id_token` is a JWT whose `sub` is the User id; with the `profile` Scope it also carries `name`, `role` and `school`. Verify its signature with the keys from [`GET /jwks`](https://id.dshs.app/docs#tag/메타데이터/GET/jwks), listed in the [discovery document](https://id.dshs.app/docs#tag/메타데이터/GET/.well-known/openid-configuration).
4. **Call the School API.** Send the access token as `Authorization: Bearer <token>`, for example to [`GET /v2/users/me`](https://api.dshs.app/docs#tag/내-정보/GET/v2/users/me). See [](dev-school-api.md).
5. **Refresh on 401.** The access token lives 15 minutes. When the School API answers `401`, post the refresh token to [`POST /token`](https://id.dshs.app/docs#tag/토큰/POST/token) and retry; see [Refresh on 401](#refresh-on-401) below.

## Code

The snippets use plain HTTP through `fetch`, with no library. Replace `timetable-web`, the redirect URI and the Scopes with the values the admin registered for your Client: `scope` always contains `openid`, adds `profile` and `offline_access` if you need them, and then the School API Scopes your app uses.

<tabs>
<tab title="Server app">

Node.js 18 or later (global `fetch`). Your server keeps the refresh token; the browser only ever sees your own session cookie.

```javascript
const ISSUER = 'https://id.dshs.app';
const SCHOOL_API = 'https://api.dshs.app'; // the School API base URL of the user's School
const CLIENT_ID = process.env.DSHS_CLIENT_ID;
const CLIENT_SECRET = process.env.DSHS_CLIENT_SECRET;
const REDIRECT_URI = 'https://timetable.example/callback'; // exactly as registered

const basicAuth = 'Basic ' + Buffer.from(
  `${encodeURIComponent(CLIENT_ID)}:${encodeURIComponent(CLIENT_SECRET)}`,
).toString('base64');

// POST /token with HTTP Basic client authentication
async function tokenRequest(params) {
  const res = await fetch(`${ISSUER}/token`, {
    method: 'POST',
    headers: {
      Authorization: basicAuth,
      'Content-Type': 'application/x-www-form-urlencoded',
    },
    body: new URLSearchParams({ client_id: CLIENT_ID, resource: SCHOOL_API, ...params }),
  });
  const body = await res.json();
  if (!res.ok) throw Object.assign(new Error(body.error), body); // { error, error_description }
  return body; // { access_token, token_type, expires_in: 900, id_token, refresh_token, scope }
}

// Step 1, your /login route: redirect the browser here.
// Generate `state` randomly per login and keep it in the user's session.
function authUrl(state) {
  const url = new URL('/auth', ISSUER);
  url.search = new URLSearchParams({
    response_type: 'code',
    client_id: CLIENT_ID,
    redirect_uri: REDIRECT_URI,
    scope: 'openid profile offline_access me.read',
    state,
  });
  return url.toString();
}

// Step 2, your /callback route: check `state` against the session, then exchange the code.
// Store the result server-side (session store or database), keyed by your own session.
async function exchangeCode(code) {
  return tokenRequest({ grant_type: 'authorization_code', code, redirect_uri: REDIRECT_URI });
}

// Used by the refresh-on-401 wrapper below
async function refresh(refreshToken) {
  return tokenRequest({ grant_type: 'refresh_token', refresh_token: refreshToken });
}
```

</tab>
<tab title="SPA">

Runs in the browser. The PKCE verifier and challenge come from the Web Crypto API; the code exchange sends `code_verifier` instead of a secret.

```javascript
const ISSUER = 'https://id.dshs.app';
const SCHOOL_API = 'https://api.dshs.app'; // the School API base URL of the user's School
const CLIENT_ID = 'timetable-web';
const REDIRECT_URI = `${location.origin}/callback`; // exactly as registered

const base64url = (bytes) =>
  btoa(String.fromCharCode(...new Uint8Array(bytes)))
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
const randomString = () => base64url(crypto.getRandomValues(new Uint8Array(32)));

// POST /token as a public Client: client_id, no secret
async function tokenRequest(params) {
  const res = await fetch(`${ISSUER}/token`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({ client_id: CLIENT_ID, resource: SCHOOL_API, ...params }),
  });
  const body = await res.json();
  if (!res.ok) throw Object.assign(new Error(body.error), body); // { error, error_description }
  return body; // { access_token, token_type, expires_in: 900, id_token, refresh_token, scope }
}

// Step 1: PKCE verifier + S256 challenge, then go to /auth
async function login() {
  const verifier = randomString(); // 43 characters
  const digest = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(verifier));
  const state = randomString();
  sessionStorage.setItem('pkce_verifier', verifier);
  sessionStorage.setItem('oauth_state', state);

  const url = new URL('/auth', ISSUER);
  url.search = new URLSearchParams({
    response_type: 'code',
    client_id: CLIENT_ID,
    redirect_uri: REDIRECT_URI,
    scope: 'openid profile offline_access me.read',
    state,
    code_challenge: base64url(digest),
    code_challenge_method: 'S256',
  });
  location.assign(url);
}

// Step 2, on the /callback page: check `state`, then exchange the code with the verifier
async function handleCallback() {
  const params = new URLSearchParams(location.search);
  if (params.has('error')) throw new Error(params.get('error'));
  if (params.get('state') !== sessionStorage.getItem('oauth_state')) throw new Error('state mismatch');

  const tokens = await tokenRequest({
    grant_type: 'authorization_code',
    code: params.get('code'),
    redirect_uri: REDIRECT_URI,
    code_verifier: sessionStorage.getItem('pkce_verifier'),
  });
  sessionStorage.removeItem('pkce_verifier');
  sessionStorage.removeItem('oauth_state');
  return tokens;
}

// Used by the refresh-on-401 wrapper below
async function refresh(refreshToken) {
  return tokenRequest({ grant_type: 'refresh_token', refresh_token: refreshToken });
}
```

</tab>
</tabs>

## Which access token you get {id="token-kind"}

With `resource=https://api.dshs.app` at `/token` you get a School API JWT (15 minutes, `aud` = that base URL, `scope` = the School API Scopes the user consented to); without `resource` you get an opaque token that only works at [`GET /userinfo`](https://id.dshs.app/docs#tag/사용자-정보/GET/userinfo). The `id_token` already carries the `profile` claims, so apps that use the School API always send `resource`; the "access token의 종류" section of [the reference](https://id.dshs.app/docs) has the details.

## Refresh on 401 {id="refresh-on-401"}

Do not track the expiry clock: when the School API answers `401`, refresh at `/token` and send the same request once more. If the refresh fails with `invalid_grant` (30 days unused, consent revoked, logged out, or an old refresh token reused), log the user in again; an app without `offline_access` has no refresh token and goes straight to login.

Every refresh returns a **new** refresh token: save it and drop the old one. This wrapper works with the `refresh()` function of either tab:

```javascript
class LoginRequired extends Error {}

// session: { accessToken, refreshToken } of the current user, from the code exchange
async function callSchoolApi(session, path, init = {}) {
  const send = () => fetch(new URL(path, SCHOOL_API), {
    ...init,
    headers: { ...init.headers, Authorization: `Bearer ${session.accessToken}` },
  });

  const res = await send();
  if (res.status !== 401) return res;
  if (!session.refreshToken) throw new LoginRequired(); // no offline_access

  try {
    const tokens = await refresh(session.refreshToken);
    session.accessToken = tokens.access_token;
    session.refreshToken = tokens.refresh_token; // rotated on every refresh
  } catch (err) {
    if (err.error === 'invalid_grant') throw new LoginRequired();
    throw err;
  }
  return send(); // retry once; a second 401 goes to the caller
}
```

Start the login again (step 1) wherever you catch `LoginRequired`.

## Logout

To log the user out of your app, delete the stored tokens and revoke the refresh token at [`POST /token/revocation`](https://id.dshs.app/docs#tag/토큰/POST/token/revocation); a School API JWT cannot be revoked and simply expires after 15 minutes. If the user also wants to sign out of dshs.app itself, send the browser to [`GET /session/end`](https://id.dshs.app/docs#tag/로그아웃/GET/session/end) with `id_token_hint` (a confirmation page appears once); returning to your app afterwards needs a post-logout redirect URI registered by the admin.

## Using a library

A library does most of this code for you (discovery, PKCE, token validation, refresh): the library table in [the reference](https://id.dshs.app/docs) lists one per environment.
