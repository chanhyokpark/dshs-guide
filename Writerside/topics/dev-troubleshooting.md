# 문제 해결

연동 중 자주 만나는 오류와 해결 방법을 모았습니다. 로그인·토큰 관련 오류는 [로그인 API 문서](https://id.dshs.app/docs)의 **오류** 표에서, API 호출 오류는 [학교 API 문서](https://api.dshs.app/docs)의 **오류** 항목에서 자세히 확인할 수 있습니다.

## redirect_uri가 일치하지 않을 때

**증상:** id.dshs.app에서 로그인 화면 대신 `invalid_redirect_uri` 오류가 표시되거나, `/token`에서 인증 코드를 교환할 때 `invalid_grant`가 반환됩니다.

**원인:** `redirect_uri`가 Client에 등록된 주소와 정확히 일치하지 않거나, `/auth`와 `/token`에 서로 다른 주소를 보냈습니다. 프로토콜, 호스트, 포트, 경로뿐 아니라 주소 끝의 `/`도 일치해야 합니다.

**해결:** 등록된 주소를 `/auth`와 `/token`에 똑같이 보내세요. 개발용 `localhost`의 포트나 도메인이 바뀌었다면 [Client 발급 안내](dev-client.md)의 방법으로 관리자에게 주소 추가를 요청하세요. [로그인 API 문서](https://id.dshs.app/docs)에서는 미등록 주소를 `invalid_request` 항목에서 설명하지만, 실제 로그인 오류 화면에는 `invalid_redirect_uri`가 표시됩니다. 코드 교환 중 발생하는 오류는 `invalid_grant` 항목을 참고하세요.

## invalid_scope

**증상:** 브라우저가 앱의 `redirect_uri`로 돌아올 때 `error=invalid_scope`가 붙어 있습니다.

**원인:** 존재하지 않거나 Client에 허용되지 않은 Scope를 요청했습니다.

**해결:** 관리자가 보내 준 목록에 있는 Scope만 요청하세요. 이름이 [Scope 목록](dev-school-api.md#scopes)과 정확히 일치하는지도 확인하세요. 권한이 더 필요하면 관리자에게 추가를 요청하면 됩니다. 자세한 설명은 [로그인 API 문서](https://id.dshs.app/docs)의 `invalid_scope` 항목을 참고하세요.

## invalid_target

**증상:** `/auth` 또는 `/token`이 `invalid_target`을 반환합니다.

**원인:** `resource`가 선택한 사용자의 학교 API 주소와 다르거나, 학교 API Scope 없이 `openid profile`만 요청해 로그인했습니다.

**해결:** `resource`에 해당 학교의 API 문서에 나온 기본 주소를 넣으세요. 대전과학고등학교는 `https://api.dshs.app`입니다. 학교 API를 사용하려면 필요한 학교 API Scope도 요청해야 합니다. 로그인 기능만 사용하는 앱은 `resource`를 생략하세요. [액세스 토큰의 종류](dev-login.md#token-kind)와 [로그인 API 문서](https://id.dshs.app/docs)의 `invalid_target` 항목을 참고하세요.

## 로그인한 지 15분쯤 지나면 401이 발생할 때

**증상:** 로그인 직후에는 되던 학교 API 호출이 약 15분 뒤부터 `401`로 실패합니다.

**원인:** 액세스 토큰이 만료되었습니다. 유효 기간은 15분입니다.

**해결:** [401 응답 시 토큰 갱신하기](dev-login.md#refresh-on-401)에 따라 토큰을 갱신한 뒤 같은 요청을 한 번 더 보내세요. [로그인 API 문서](https://id.dshs.app/docs)의 학교 API `401` 항목에서도 확인할 수 있습니다.

## 토큰 갱신 중 invalid_grant가 발생할 때

**증상:** `/token`에 토큰 갱신을 요청하면 `400 invalid_grant`가 반환됩니다.

**원인:** 리프레시 토큰을 30일 동안 사용하지 않았거나, 사용자가 동의를 철회했거나 로그아웃했을 수 있습니다. 한 번의 로그인으로 유지할 수 있는 최대 기간인 1년이 지난 경우에도 발생합니다. 갱신할 때마다 새 리프레시 토큰이 발급되므로, 이전 토큰을 유예 시간인 5분이 지난 뒤 다시 사용해도 이 오류가 발생합니다.

**해결:** 로그인을 다시 시작하세요. 이후에는 갱신 응답으로 받은 최신 리프레시 토큰을 저장하고, 이전 토큰은 사용하지 마세요. 자세한 설명은 [로그인 API 문서](https://id.dshs.app/docs)의 `invalid_grant` 항목을 참고하세요.

## insufficient_scope

**증상:** 학교 API가 `403`을 반환하며, 본문에 `message: "insufficient_scope"`가 들어 있습니다.

**원인:** 해당 API의 **필요한 Scope**에 나와 있는 개별 권한이 토큰에 없습니다. 부족한 권한은 `WWW-Authenticate` 헤더의 `scope` 값으로 확인할 수 있습니다.

**해결:** `/auth`의 `scope`에 필요한 권한을 추가하고, 사용자가 다시 로그인해 동의하도록 하세요. Client에 아직 허용되지 않은 Scope라면 먼저 관리자에게 추가를 요청해야 합니다.

같은 `403`이라도 `forbidden`은 사용자의 역할이나 해당 항목에 대한 권한 때문에 거부된 경우입니다. 예를 들어 다른 사람이 쓴 글을 수정하려는 경우에는 Scope를 추가해도 해결되지 않습니다. [학교 API 문서](https://api.dshs.app/docs)의 `403` 항목을 참고하세요.

## 사용자 정보에 id만 있을 때

**증상:** 응답의 다른 사용자 정보에 `{ "id": "…" }`만 있고, 이름이나 좌석 정보가 없습니다.

**원인:** 토큰에 `user.read`가 없습니다. `user.detail.read`가 없으면 학년·반·번호 같은 상세 정보도 제외됩니다.

**해결:** `user.read`를 요청하고, 상세 정보가 필요하면 `user.detail.read`도 함께 요청하세요. 다만 조회하는 사용자의 역할에 따라 비공개 처리된 정보는 이 권한이 있어도 제공되지 않습니다. [Scope 안내](dev-school-api.md#scopes)를 참고하세요.

## 요청이 너무 많을 때 (429)

**증상:** 학교 API가 `429`를 반환하며, 본문에 `message: "rate_limited"`가 들어 있습니다.

**원인:** Client의 요청이 60초에 120회를 넘었습니다. 같은 Client를 사용하는 모든 사용자의 요청을 합산한 횟수입니다.

**해결:** `Retry-After`에 표시된 시간(초)만큼 기다린 뒤 다시 요청하세요. 응답을 캐시해 중복 호출을 줄이고, `X-RateLimit-Remaining`으로 남은 횟수를 확인하세요. [요청 횟수 제한](dev-school-api.md#rate-limit)과 [학교 API 문서](https://api.dshs.app/docs)의 `429` 항목을 참고하세요.

## 브라우저에서 CORS 오류가 발생할 때

**증상:** api.dshs.app이나 id.dshs.app으로 보낸 요청이 차단되고, 브라우저 콘솔에 CORS 오류가 표시됩니다.

**원인:** api.dshs.app은 모든 출처(origin)에서의 요청을 허용하지만 쿠키 인증은 허용하지 않습니다. 따라서 `credentials: 'include'`로 요청하면 실패합니다. id.dshs.app의 토큰 API는 SPA·네이티브 앱으로 등록한 Client에 한해, 등록된 redirect URI와 같은 출처에서 보낸 브라우저 요청만 허용합니다.

**해결:** 학교 API를 호출할 때는 쿠키 없이 Bearer 헤더로 토큰을 보내세요. 서버 앱은 서버에서 인증 코드를 교환하고, SPA는 등록된 redirect URI와 같은 출처에서 실행하세요. [로그인 연동 안내](dev-login.md)를 참고하세요.

## 사용자가 동의를 철회했을 때

**증상:** 토큰 갱신이 `invalid_grant`로 실패하고, 다음 로그인 때 동의 화면이 다시 표시됩니다.

**원인:** 사용자가 [앱 권한 관리](https://id.dshs.app/consents)에서 앱에 대한 동의를 철회했습니다. 해당 사용자와 Client에 연결된 리프레시 토큰은 모두 무효가 됩니다.

**해결:** [이용 규칙](developer.md#usage-rules)에 따라 해당 동의로 수집해 저장한 데이터를 삭제하세요. 사용자가 앱에 다시 방문하면 로그인을 새로 진행합니다. 자세한 설명은 [로그인 API 문서](https://id.dshs.app/docs)의 `invalid_grant` 항목을 참고하세요.

## user_locked 또는 user_deactivated

**증상:** 학교 API가 `user_locked` 또는 `user_deactivated`와 함께 `401`을 반환합니다. `/token`에서는 `invalid_grant`가 반환되고, `error_description`에 이 값이 들어갑니다.

**원인:** 학교에서 해당 사용자를 잠그거나 비활성화했습니다.

**해결:** 재시도를 멈추고 사용자에게 학교로 문의하도록 안내하세요. 잠금이나 비활성화 상태가 해제되기 전에는 다시 로그인해도 해결되지 않습니다. [로그인 API 문서](https://id.dshs.app/docs)의 `invalid_grant` 항목과 [학교 API 문서](https://api.dshs.app/docs)의 `401` 항목을 참고하세요.
