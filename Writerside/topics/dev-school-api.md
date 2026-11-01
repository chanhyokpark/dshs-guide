# Calling the School API

The School API is a JSON API over HTTPS. Its base URL is per School (`https://api.dshs.app` for 대전과학고등학교), and every call that needs a user carries the School API access token from [](dev-login.md) as `Authorization: Bearer <token>`. [api.dshs.app/docs](https://api.dshs.app/docs) lists every route with its Scopes and schemas.

## Scopes {id="scopes"}

A Scope name is a tree path, `resource.subresource.action`, with the action (`read`, `write`, `submit`, `download`) always last: a leaf (잎) Scope such as `board.post.read` allows one function, and an umbrella (우산) Scope, the bare resource name such as `board`, includes every leaf below it. Each route lists the leaves it needs under 필요한 Scope and answers `403 insufficient_scope` when one is missing, with one exception: routes that return other Users also list `user.read`, and without it they still work but each User comes back as `{ "id": "…" }` only (`user.read` plus `user.detail.read` adds details such as grade, class and number). A Scope never goes beyond what the logged-in user can already do in dshs.app.

The eight umbrella Scopes, with the label users see on the consent page (동의 화면):

| Scope | Label (동의 화면) | English gloss | Opens |
|---|---|---|---|
| `me` | 모든 내 정보 기능 사용 | All of "my profile" | `me.read`, `me.phone.read` |
| `user` | 모든 다른 사용자 정보 기능 사용 | All of "other users' info" | `user.read`, `user.detail.read` |
| `dorm` | 모든 기숙사 신청 기능 사용 | All of "dormitory requests" | `dorm.read`, `dorm.write` |
| `request` | 모든 신청 기능 사용 | All of "requests and applications" | `request.read`, `request.write` |
| `board` | 모든 게시판 기능 사용 | All of "boards" | `board.read`, `board.write`, `board.post.*`, `board.assignment.*`, and the lost and found, wake-up song and student council boards (`board.lost_and_found.*`, `board.song_request.*`, `board.student_council.*`) |
| `form` | 모든 설문 기능 사용 | All of "surveys" | `form.read`, `form.response.write` |
| `attachment` | 모든 첨부파일 기능 사용 | All of "attachments" | `attachment.read`, `attachment.write`, `attachment.download` |
| `oj` | 모든 온라인 저지 기능 사용 | All of "online judge" | `oj.me.*`, `oj.problem.read`, `oj.submission.*`, `oj.set.*` |

`reservation.read`, `penalty.read` and `message.read` have no umbrella; request them by name. Every leaf with its label is in the **dshs** entry of the Authentication panel on [api.dshs.app/docs](https://api.dshs.app/docs).

## First-party only

Some functions stay inside the dshs.app apps and have no Scope at all: study seat reservation and cancellation (자습 좌석 신청), the legacy out-request module ([외출·특별실 신청](out.md)), online judge contests, notifications (알림), sending messages and authoring forms. 일반 신청 forms, including 외출 and 특별실 forms, are reachable through the `request` Scope. A route that is not in the reference cannot be called with an external app's token.

## Response rules

- Paginated lists return `{ items, total, page, perPage, totalPages }`; `page` starts at 1 and `perPage` sets the page size (default and maximum per route).
- Paths mostly use the short `uid`; Users are identified by `id`. Online judge problems use the problem number and messages an integer `id`.
- A field without a value keeps its key with `null`. Only fields that need a Scope you lack are left out.
- Times are ISO 8601 in UTC (`2026-03-02T00:30:00.000Z`). Dates are mostly `YYYY-MM-DD`; meal keys and the seat and dormitory routes use `YYYYMMDD`.
- An attachment's `downloadUrl` is relative to the School API base URL, and the download needs the `Authorization: Bearer` header too.
- Errors look like `{ "statusCode": 403, "message": "insufficient_scope", "error": "Forbidden" }`, with a snake_case code in `message`.

The 응답 규칙 and 오류 sections of [api.dshs.app/docs](https://api.dshs.app/docs) spell out each rule.

## Rate Limit {id="rate-limit"}

Each Client may send **120 requests per 60 seconds**, counted across all users of that Client; calls without a token are not counted.
Responses carry `X-RateLimit-Limit`, `X-RateLimit-Remaining` and `X-RateLimit-Reset` (Unix time in seconds).
Over the limit you get `429` (`rate_limited`) with `Retry-After` in seconds; see the Rate Limit section of [the reference](https://api.dshs.app/docs).

## Read a paginated list

This reads every page of [`GET /v2/messages/threads`](https://api.dshs.app/docs#tag/메시지/GET/v2/messages/threads) (Scopes `message.read` and `user.read`). In your app, send the request through `callSchoolApi()` from [Refresh on 401](dev-login.md#refresh-on-401) so an expired token is refreshed for you.

```javascript
const SCHOOL_API = 'https://api.dshs.app'; // the School API base URL, as on the login page

async function listAllThreads(accessToken) {
  const threads = [];
  for (let page = 1; ; page++) {
    const url = new URL('/v2/messages/threads', SCHOOL_API);
    url.search = new URLSearchParams({ page: String(page), perPage: '50' });

    const res = await fetch(url, { headers: { Authorization: `Bearer ${accessToken}` } });
    if (!res.ok) throw new Error(`${res.status} ${(await res.json()).message}`);

    const { items, totalPages } = await res.json();
    threads.push(...items);
    if (page >= totalPages) return threads;
  }
}
```

## Try it in the reference {id="try-it"}

[api.dshs.app/docs](https://api.dshs.app/docs) can send real requests for you:

- **SPA·네이티브 앱 Clients**: in Authentication choose **dshs**, enter your `client_id`, pick Scopes and log in. The admin must first add `https://api.dshs.app/docs` to your Client's redirect URIs ([](dev-client.md#try-it-redirect)).
- **Server apps**: get an access token with your own code and paste it into **Bearer Token**.
