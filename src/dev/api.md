# API
dshs.app에는 자습 신청, 벌점 확인, 외출 신청, 급식 조회 등을 지원하는 REST API가 구현되어 있습니다.

## API 키 발급
API를 사용하려면 API 키를 받아야 합니다. 
1. 피드백 페이지에서 다음 정보를 제출합니다:
   + API의 사용 목적
   + 클라이언트의 이름 및 설명
   + 리다이렉트 URL 목록
   + 필요한 권한 목록
2. 요청이 승인되면 Client ID 및 Client Secret이 이메일로 발송됩니다.

## 인증 방법
급식 조회를 제외한 모든 API는 OAuth 2.0 인증 방식 중 Authorization Code Grant 방식만을 지원합니다.

Access Token은 발급받은 뒤 30일간 유효하며, Refresh Token은 지원하지 않습니다.

### 1. 사용자 인증 요청
```http request
GET https://www.dshs.app/authorize
```
매개변수

| 이름           | 필수 | 설명             |
|--------------|----|----------------|
| client_id    | 필수 | 발급받은 Client ID |
| redirect_uri | 필수 | 인증 코드를 받을 URL  |
| state        | 선택 | CSRF 방지용 문자열   |


인증 코드는 `GET {REDIRECT_URI}?code={CODE}` 형식으로 전달됩니다.

### 2. Access Token 발급
```http request
POST https://www.dshs.app/api/v1/token
Content-Type: application/x-www-form-urlencoded
```
매개변수

| 이름            | 필수 | 설명                           |
|---------------|----|------------------------------|
| grant_type    | 필수 | 반드시 "authorization_code"여야 함 |
| client_id     | 필수 | 발급받은 Client ID               |
| client_secret | 필수 | Client Secret                |
| code          | 필수 | 인증 코드                        |

응답
```json
{
  "access_token": "{ACCESS_TOKEN}",
  "token_type": "Bearer",
  "expires_in": 2592000, 
  "scope": "{SCOPE}"
}
```

## API 호출
모든 API 요청에는 발급받은 Access Token을 Authorization 헤더에 포함해야 합니다.

예시
```http request
GET https://www.dshs.app/api/v1/userinfo
Authorization: Bearer {ACCESS_TOKEN}
```
## API 목록
[API 문서](https://www.dshs.app/api-doc)를 참고하세요.

