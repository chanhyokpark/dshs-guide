# 학교 API 사용하기

학교 API는 HTTPS로 요청하고 JSON으로 응답을 받습니다. 기본 주소는 학교마다 다르며, 대전과학고등학교는 `https://api.dshs.app`을 사용합니다. 사용자 인증이 필요한 요청에는 [로그인 연동](dev-login.md) 과정에서 받은 학교 API용 액세스 토큰을 `Authorization: Bearer <token>` 헤더에 넣으세요. 전체 경로와 필요한 Scope, 요청·응답 형식은 [학교 API 문서](https://api.dshs.app/docs)에서 확인할 수 있습니다.

## 권한(Scope) {id="scopes"}

Scope는 앱에 허용할 기능의 범위를 나타냅니다. 개별 Scope는 `자원.하위자원.동작` 형태로 이름을 붙이며, 마지막에 `read`, `write`, `submit`, `download` 같은 동작이 옵니다.

- **개별(leaf) Scope**는 특정 기능을 허용합니다. 예를 들어 `board.post.read`는 게시글 읽기 권한입니다.
- **통합(umbrella) Scope**는 해당 자원의 모든 하위 권한을 포함합니다. 예를 들어 `board`를 요청하면 게시판에 속한 개별 Scope를 모두 요청한 것과 같습니다.

각 API 설명의 **필요한 Scope**에 있는 권한이 하나라도 없으면 `403 insufficient_scope` 오류가 발생합니다. 다만 다른 사용자 정보를 조회하는 `user.read`는 예외입니다. 이 Scope가 없어도 API는 호출할 수 있지만, 응답의 사용자 정보에는 `{ "id": "…" }`만 남습니다. 이름·학번·좌석 정보만 따로 제공하는 필드도 제외됩니다. 학년·반·번호 같은 상세 정보까지 받으려면 `user.read`와 `user.detail.read`가 모두 필요합니다.

Scope를 허용받아도 **로그인한 사용자가 원래 할 수 있는 일의 범위를 넘을 수는 없습니다**.

통합 Scope는 다음 여덟 가지입니다. 동의 화면에는 표에 적힌 문구가 표시됩니다.

| Scope | 동의 화면의 문구 | 포함하는 권한 |
|---|---|---|
| `me` | 모든 내 정보 기능 사용 | `me.read`, `me.phone.read` |
| `user` | 모든 다른 사용자 정보 기능 사용 | `user.read`, `user.detail.read` |
| `dorm` | 모든 기숙사 신청 기능 사용 | `dorm.read`, `dorm.write` |
| `request` | 모든 신청 기능 사용 | `request.read`, `request.write` |
| `board` | 모든 게시판 기능 사용 | `board.read`, `board.write`, `board.post.*`, `board.assignment.*`, 분실물·기상송·학생회 게시판 권한(`board.lost_and_found.*`, `board.song_request.*`, `board.student_council.*`) |
| `form` | 모든 설문 기능 사용 | `form.read`, `form.response.write` |
| `attachment` | 모든 첨부파일 기능 사용 | `attachment.read`, `attachment.write`, `attachment.download` |
| `oj` | 모든 온라인 저지 기능 사용 | `oj.me.*`, `oj.problem.read`, `oj.submission.*`, `oj.set.*` |

`reservation.read`, `penalty.read`, `message.read`는 통합 Scope가 없으므로 각각 요청해야 합니다. 개별 Scope의 전체 목록과 설명은 [학교 API 문서](https://api.dshs.app/docs)의 **Authentication → dshs**에서 확인하세요.

## dshs.app 자체 앱에서만 제공하는 기능

자습 좌석 신청·취소, 기존 [외출·특별실 신청](out.md), 온라인 저지 대회, 알림, 메시지 발송, 설문 작성은 외부 앱에 제공하지 않습니다. 이 기능에 해당하는 Scope도 없습니다.

다만 **일반 신청 양식으로 만든 외출·특별실 신청**은 `request` Scope로 사용할 수 있습니다. 기존 외출·특별실 신청 기능과 구분해 주세요. API 문서에 없는 경로는 외부 앱의 토큰으로 호출할 수 없습니다.

## 응답 규칙

- 페이지로 나뉜 목록은 `{ items, total, page, perPage, totalPages }` 형식으로 반환됩니다. `page`는 1부터 시작하며, `perPage`는 페이지당 항목 수입니다. 기본값과 최댓값은 API마다 다릅니다.
- 경로에서는 대부분 짧은 `uid`를 사용합니다. 사용자는 `id`, 온라인 저지 문제는 문제 번호, 메시지는 정수 `id`로 구분합니다.
- 값이 없는 필드는 키를 유지하고 `null`을 반환합니다. 특정 Scope가 필요한 필드는 해당 Scope가 없으면 응답에서 제외됩니다.
- 시각은 ISO 8601 UTC 형식(`2026-03-02T00:30:00.000Z`)입니다. 날짜는 대부분 `YYYY-MM-DD` 형식이지만, 급식의 날짜 키와 자습 좌석·기숙사 API는 `YYYYMMDD`를 사용합니다.
- 첨부파일의 `downloadUrl`은 학교 API 기본 주소를 기준으로 한 상대 경로입니다. 파일을 내려받는 요청에도 `Authorization: Bearer` 헤더가 필요합니다.
- 오류 응답은 `{ "statusCode": 403, "message": "insufficient_scope", "error": "Forbidden" }` 형식이며, `message`에 snake_case 오류 코드가 담깁니다. 요청 검증에 실패한 `400` 응답에서는 `message`가 문자열 배열입니다.

자세한 내용은 [학교 API 문서](https://api.dshs.app/docs)의 **응답 규칙**과 **오류**를 참고하세요.

## 요청 횟수 제한 {id="rate-limit"}

Client 하나당 **60초에 120회**까지 요청할 수 있습니다. 같은 Client로 로그인한 모든 사용자의 요청을 합산하며, 토큰 없이 호출하는 API는 집계에서 제외합니다.

응답의 `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` 헤더에서 허용 횟수, 남은 횟수, 제한이 초기화되는 시각(Unix 타임스탬프, 초)을 확인할 수 있습니다. 제한을 넘으면 `429`(`rate_limited`) 응답을 받으며, `Retry-After` 헤더에 다시 요청하기까지 기다려야 할 시간(초)이 표시됩니다. 자세한 내용은 [학교 API 문서](https://api.dshs.app/docs)의 **Rate Limit**을 참고하세요.

## 여러 페이지의 목록 읽기

아래 예시는 [`GET /v2/messages/threads`](https://api.dshs.app/docs#tag/메시지/GET/v2/messages/threads)로 메시지 대화 목록을 마지막 페이지까지 읽습니다. `message.read`가 필요하며, 상대방 정보도 받으려면 `user.read`를 함께 요청하세요. 실제 앱에서는 [401 응답 시 토큰 갱신하기](dev-login.md#refresh-on-401)의 `callSchoolApi()`로 요청을 보내면 토큰이 만료되었을 때 갱신할 수 있습니다.

```javascript
const SCHOOL_API = 'https://api.dshs.app'; // 로그인 연동에 사용한 학교 API 기본 주소

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

## API 문서에서 호출해 보기 {id="try-it"}

[학교 API 문서](https://api.dshs.app/docs)에서 실제 API를 호출해 볼 수 있습니다.

- **SPA·네이티브 앱**: **Authentication → dshs**에서 `client_id`를 입력하고 Scope를 선택한 뒤 로그인하세요. 먼저 관리자에게 `https://api.dshs.app/docs`를 Client의 redirect URI로 등록해 달라고 요청해야 합니다. [설정 방법](dev-client.md#try-it-redirect)을 참고하세요.
- **서버 앱**: 앱에서 발급받은 액세스 토큰을 **Bearer Token**에 붙여 넣으세요.
