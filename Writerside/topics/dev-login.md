# dshs.app 로그인 연동

**dshs.app 계정으로 로그인**은 `https://id.dshs.app`에서 제공하며, OpenID Connect의 인증 코드 방식(Authorization Code flow)을 따릅니다. 이 페이지에서는 앱 종류에 따른 연동 방법과 코드 예제를 안내합니다. 파라미터, 토큰, 오류에 대한 자세한 설명은 [로그인 API 문서](https://id.dshs.app/docs)를 참고하세요.

## 앱 종류 선택

- **서버 앱**(confidential Client): 서버가 `client_secret`으로 인증하고, 인증 코드를 토큰으로 교환합니다. PKCE는 선택 사항이지만 사용을 권장합니다.
- **SPA·네이티브 앱**(public Client): 브라우저나 기기에서 인증 코드를 토큰으로 교환합니다. 비밀 키를 사용하지 않으므로 PKCE(S256)가 필수입니다.

백엔드가 있다면 서버 앱으로 등록하세요. 동의 화면이 표시되는 시점 등 나머지 차이는 [로그인 API 문서](https://id.dshs.app/docs)의 **Client 종류** 표에서 확인할 수 있습니다.

## 로그인부터 API 호출까지

1. **로그인 화면으로 이동합니다.** `client_id`, `redirect_uri`, `scope`, `state`를 넣은 [`GET /auth`](https://id.dshs.app/docs#tag/로그인/GET/auth) 주소로 사용자의 브라우저를 보냅니다. 사용자는 id.dshs.app에서 로그인하고, 계정에 연결된 사용자 중 하나를 선택한 뒤 앱이 요청한 권한에 동의합니다.
2. **인증 코드를 토큰으로 교환합니다.** 로그인이 끝나면 브라우저가 `code`와 `state`를 받아 앱의 `redirect_uri`로 돌아옵니다. `code`는 60초 동안 유효하며 한 번만 사용할 수 있습니다. 이 코드를 [`POST /token`](https://id.dshs.app/docs#tag/토큰/POST/token)에 보내세요. 서버 앱은 `client_secret`으로 인증하고, PKCE를 사용하는 앱은 `code_verifier`를 함께 보냅니다. 학교 API를 사용하려면 `resource=https://api.dshs.app`도 넣으세요. 응답으로 `access_token`, `id_token`을 받고, `offline_access`를 요청해 동의받았다면 `refresh_token`도 받습니다.
3. **로그인한 사용자를 확인합니다.** `id_token`은 JWT 형식이며, `sub`에 사용자 id가 담겨 있습니다. `profile` Scope에 동의했다면 `name`, `role`, `school`도 포함됩니다. 토큰은 [`GET /jwks`](https://id.dshs.app/docs#tag/메타데이터/GET/jwks)의 공개 키로 서명을 검증한 뒤 사용하세요. 공개 키의 주소는 [discovery 문서](https://id.dshs.app/docs#tag/메타데이터/GET/.well-known/openid-configuration)에도 나와 있습니다.
4. **학교 API를 호출합니다.** `Authorization: Bearer <token>` 헤더에 액세스 토큰을 넣습니다. 예를 들어 [`GET /v2/users/me`](https://api.dshs.app/docs#tag/내-정보/GET/v2/users/me)로 내 정보를 조회할 수 있습니다. 자세한 방법은 [학교 API 사용하기](dev-school-api.md)를 참고하세요.
5. **토큰이 만료되면 갱신합니다.** 액세스 토큰의 유효 기간은 15분입니다. 학교 API가 `401`을 반환하면 [`POST /token`](https://id.dshs.app/docs#tag/토큰/POST/token)에 리프레시 토큰을 보내 새 토큰을 발급받고, 원래 요청을 다시 보냅니다. 아래의 [401 응답 시 토큰 갱신하기](#refresh-on-401)를 참고하세요.

## 코드 예제

아래 예제는 별도 라이브러리 없이 `fetch`로 요청을 보냅니다. `timetable-web`, redirect URI, Scope는 관리자에게 등록한 값으로 바꾸세요. `scope`에는 `openid`를 반드시 포함하고, 필요에 따라 `profile`, `offline_access`, 앱에서 사용할 학교 API Scope를 추가합니다.

<tabs>
<tab title="서버 앱">

전역 `fetch`를 지원하는 Node.js 18 이상에서 실행할 수 있습니다. 리프레시 토큰은 서버에 보관하고, 브라우저에는 앱의 세션 쿠키만 전달합니다.

```javascript
const ISSUER = 'https://id.dshs.app';
const SCHOOL_API = 'https://api.dshs.app'; // 사용자가 속한 학교의 API 기본 주소
const CLIENT_ID = process.env.DSHS_CLIENT_ID;
const CLIENT_SECRET = process.env.DSHS_CLIENT_SECRET;
const REDIRECT_URI = 'https://timetable.example/callback'; // 등록한 주소와 정확히 일치해야 합니다.

const basicAuth = 'Basic ' + Buffer.from(
  `${encodeURIComponent(CLIENT_ID)}:${encodeURIComponent(CLIENT_SECRET)}`,
).toString('base64');

// HTTP Basic으로 Client를 인증하고 POST /token을 호출합니다.
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

// 1단계: 앱의 /login에서 이 주소로 브라우저를 이동시킵니다.
// state는 로그인할 때마다 무작위로 생성해 사용자의 세션에 저장하세요.
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

// 2단계: /callback에서 state가 세션에 저장한 값과 일치하는지 확인한 뒤 코드를 교환합니다.
// 받은 토큰은 앱의 세션에 연결해 서버의 세션 저장소나 데이터베이스에 보관하세요.
async function exchangeCode(code) {
  return tokenRequest({ grant_type: 'authorization_code', code, redirect_uri: REDIRECT_URI });
}

// 아래의 callSchoolApi()에서 401 응답을 받았을 때 호출합니다.
async function refresh(refreshToken) {
  return tokenRequest({ grant_type: 'refresh_token', refresh_token: refreshToken });
}
```

</tab>
<tab title="SPA">

브라우저에서 실행하는 예제입니다. Web Crypto API로 PKCE에 필요한 verifier와 challenge를 만들고, 인증 코드를 교환할 때 비밀 키 대신 `code_verifier`를 보냅니다.

```javascript
const ISSUER = 'https://id.dshs.app';
const SCHOOL_API = 'https://api.dshs.app'; // 사용자가 속한 학교의 API 기본 주소
const CLIENT_ID = 'timetable-web';
const REDIRECT_URI = `${location.origin}/callback`; // 등록한 주소와 정확히 일치해야 합니다.

const base64url = (bytes) =>
  btoa(String.fromCharCode(...new Uint8Array(bytes)))
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
const randomString = () => base64url(crypto.getRandomValues(new Uint8Array(32)));

// public Client는 비밀 키 없이 client_id를 넣어 POST /token을 호출합니다.
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

// 1단계: PKCE verifier와 S256 challenge를 만든 뒤 /auth로 이동합니다.
async function login() {
  const verifier = randomString(); // 43자
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

// 2단계: /callback에서 state를 확인하고, verifier와 함께 코드를 보내 토큰으로 교환합니다.
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

// 아래의 callSchoolApi()에서 401 응답을 받았을 때 호출합니다.
async function refresh(refreshToken) {
  return tokenRequest({ grant_type: 'refresh_token', refresh_token: refreshToken });
}
```

</tab>
</tabs>

## 액세스 토큰의 종류 {id="token-kind"}

`/token`을 호출할 때 `resource`를 지정했는지에 따라 발급되는 액세스 토큰이 달라집니다.

- **`resource=https://api.dshs.app`을 지정한 경우**: 학교 API용 JWT를 받습니다. 유효 기간은 15분이며, `aud`에는 학교 API 주소가, `scope`에는 사용자가 동의한 학교 API 권한이 담깁니다.
- **`resource`를 생략한 경우**: [`GET /userinfo`](https://id.dshs.app/docs#tag/사용자-정보/GET/userinfo)에서만 사용할 수 있는 불투명(opaque) 토큰을 받습니다.

학교 API를 사용하는 앱은 `resource`를 지정하세요. `profile` Scope에 동의했다면 사용자 정보는 `id_token`에도 포함되므로 보통 `/userinfo`를 따로 호출할 필요가 없습니다. 자세한 내용은 [로그인 API 문서](https://id.dshs.app/docs)의 **access token의 종류**를 참고하세요.

## 401 응답 시 토큰 갱신하기 {id="refresh-on-401"}

만료 시각을 따로 추적할 필요 없이, 학교 API가 `401`을 반환하면 `/token`에서 토큰을 갱신한 뒤 같은 요청을 한 번 더 보내면 됩니다. 갱신도 `invalid_grant`로 실패하면 다시 로그인해야 합니다. 30일 동안 사용하지 않았거나, 동의 철회·로그아웃·이전 리프레시 토큰 재사용 등으로 토큰이 무효가 된 경우입니다. `offline_access`를 요청하지 않은 앱은 리프레시 토큰이 없으므로 바로 로그인을 다시 시작합니다.

갱신할 때마다 **새 리프레시 토큰**이 발급됩니다. 새 토큰을 저장하고 이전 토큰은 삭제하세요. 아래 함수는 앞의 서버 앱 또는 SPA 예제에 있는 `refresh()`와 함께 사용합니다.

```javascript
class LoginRequired extends Error {}

// session: 코드 교환으로 받은 현재 사용자의 토큰 { accessToken, refreshToken }
async function callSchoolApi(session, path, init = {}) {
  const send = () => {
    const headers = new Headers(init.headers); // Headers 인스턴스와 일반 객체 모두 처리합니다.
    headers.set('Authorization', `Bearer ${session.accessToken}`);
    return fetch(new URL(path, SCHOOL_API), { ...init, headers });
  };

  const res = await send();
  if (res.status !== 401) return res;
  if (!session.refreshToken) throw new LoginRequired(); // offline_access를 받지 않은 경우

  try {
    const tokens = await refresh(session.refreshToken);
    session.accessToken = tokens.access_token;
    session.refreshToken = tokens.refresh_token; // 갱신할 때마다 새 값으로 교체합니다.
    // 여기서 변경한 세션을 세션 저장소나 데이터베이스에도 저장하세요.
  } catch (err) {
    if (err.error === 'invalid_grant') throw new LoginRequired();
    throw err;
  }
  return send(); // 한 번만 재시도합니다. 다시 401을 받으면 호출한 쪽에서 처리합니다.
}
```

`LoginRequired`를 잡은 곳에서 1단계의 로그인 절차를 다시 시작하세요.

## 로그아웃

앱에서 로그아웃할 때는 저장한 토큰을 삭제하고, [`POST /token/revocation`](https://id.dshs.app/docs#tag/토큰/POST/token/revocation)으로 리프레시 토큰을 무효화하세요. 학교 API용 JWT는 개별적으로 무효화할 수 없으며, 발급 후 15분이 지나면 만료됩니다.

사용자가 dshs.app에서도 로그아웃하길 원하면 `id_token_hint`를 넣어 브라우저를 [`GET /session/end`](https://id.dshs.app/docs#tag/로그아웃/GET/session/end)로 보냅니다. 확인 화면이 한 번 표시됩니다. 로그아웃 후 앱으로 돌아오게 하려면 관리자에게 로그아웃 후 돌아올 주소도 미리 등록해야 합니다.

<note>

OIDC 라이브러리를 사용하면 서버 설정 조회(discovery), PKCE, 토큰 검증·갱신을 대부분 처리해 줍니다. [로그인 API 문서](https://id.dshs.app/docs)의 라이브러리 표에서 개발 환경에 맞는 도구를 확인하세요.

</note>
