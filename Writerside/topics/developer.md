# Developers

Current students and staff can build their own apps on DSHS.APP data. This section is the short path from idea to first API call; the two reference pages below stay the full and canonical description.

## What you get

- **Log in with dshs.app** (dshs.app 계정으로 로그인): users sign in on `id.dshs.app` and come back to your app with tokens, like "Sign in with Google". It is standard OpenID Connect, and your app never sees a password.
- **School API** (학교 API): read and write the logged-in user's school data (profile, boards, requests, dormitory, online judge and more) with the access token from that login.

One **Client** (an app registered with the admin) covers both.

## Who can request a Client

Current students and staff (재학생과 교직원) can request a Client. The admin reviews every request and may refuse it.

## Reference

| Reference | What it covers |
|---|---|
| [id.dshs.app/docs](https://id.dshs.app/docs) | Login, tokens, consent, logout. One login server for every School. |
| [api.dshs.app/docs](https://api.dshs.app/docs) | The School API of 대전과학고등학교: every route, its Scopes, request and response schemas. |

The School API base URL is **per School**: `https://api.dshs.app` is the base URL for 대전과학고등학교, and a second School gets its own.

<warning>

**Beta.** Both APIs are in beta. Routes and responses may still change; the admin tells registered developers about changes directly.

</warning>

## Usage rules {id="usage-rules"}

<warning>

Every app that uses a Client must follow these rules (이용 규칙).

1. Use the API only for the purpose and scope you stated when you requested the Client.
2. Do not automate first-party-only functions such as study seat reservation (자습 좌석 신청).
3. Do not scrape the dshs.app web app.
4. Respect the [Rate Limit](dev-school-api.md#rate-limit).
5. Store only data you received through the Scopes the user consented to, and delete it when the user revokes consent.
6. Do not re-share other students' personal data.

</warning>

## Get started

1. [](dev-client.md): ask the admin for a Client.
2. [](dev-login.md): add the login to your app and get tokens.
3. [](dev-school-api.md): call the School API with the access token.
4. [](dev-troubleshooting.md): look up an error you hit on the way.
