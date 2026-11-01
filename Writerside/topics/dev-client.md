# Getting a Client

A **Client** is your app as registered on the login server: it has a `client_id`, the redirect URIs users may return to, and the Scopes it may request. There is no self-service registration page; the admin registers every Client by hand after reading your request.

## Request a Client

Send the request through the private contact flow, not through a public GitHub issue: redirect URIs and secrets must not be public.

1. Open the [DSHS.APP contact page](https://dshs.app/contact) and complete the human check.
2. Press **문의 이메일 보기** and write to the address it shows.
3. Copy the template below into the email and fill in every row.

| Field | What to write | Example |
|---|---|---|
| 앱 이름 (app name) | The name users see on the consent page (동의 화면). | 시간표 도우미 |
| 사용 목적 (purpose) | What the app does and who uses it. The [usage rules](developer.md#usage-rules) hold you to this purpose. | Shows my class timetable together with assignments due this week. |
| 앱 종류 (app type) | 서버 앱 (server app, the code exchange runs on your server) or SPA·네이티브 앱 (SPA or native app, the code exchange runs in the browser or on the device). See [](dev-login.md) if unsure. | SPA |
| redirect URI 목록 (redirect URIs) | Every address users return to after login, exactly as your app sends it, including the development `localhost` one. Add `https://api.dshs.app/docs` if you want to try the API in the reference (SPA·네이티브 앱 only), and a post-logout URI if you use dshs.app logout. | `https://timetable.example/callback`, `http://localhost:5173/callback`, `https://api.dshs.app/docs` |
| 필요한 Scope (Scopes) | Umbrella (우산) or leaf (잎) Scope names from [the Scope list](dev-school-api.md#scopes). Ask only for what the app uses. | `me.read`, `board.post.read`, `user.read` |
| 연락처 (contact) | How the admin reaches you about this Client. | Name, student number, email |

## What comes back

The admin replies with:

- `client_id`: public, goes into every login request.
- `client_secret`: server apps only. It authenticates your server at the token endpoint.
- The list of Scopes your Client may request. Requesting anything else fails with `invalid_scope`.

To change any of this later (a new redirect URI, more Scopes), send another request the same way.

## Keep the secret on the server

Never put `client_secret` into browser code, a mobile app bundle or a Git repository; read it from an environment variable or a secret store on your server. If it leaks, ask the admin through the same contact flow to rotate it, then deploy the new value.

## Redirect URI for trying the API {id="try-it-redirect"}

The School API reference can log you in with your own Client and call routes for you. This works for SPA·네이티브 앱 Clients only: ask the admin to add `https://api.dshs.app/docs` as an extra redirect URI. Server apps paste an access token instead; see [](dev-school-api.md#try-it).
