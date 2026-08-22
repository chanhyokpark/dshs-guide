# 수탁자·국외이전 현황

개인정보 처리 위탁과 국외 이전 현황입니다. [개인정보 처리방침](https://dshs.app/privacy) 제7조·제8조의 근거
자료이며, 수탁자 변경 시 방침을 먼저 개정한 뒤 이 문서를 갱신합니다.

## 국내 위탁

| 수탁자 | 업무 | 전달 항목 | 리전 | 계약·약관 |
|---|---|---|---|---|
| Amazon Web Services, Inc. | 첨부파일 저장(S3), 이메일 발송(SES), 비동기 작업(SQS) | 첨부파일, 수신자 이메일, 작업 데이터 | 국내(서울) — 국외 이전 없음 | [AWS DPA](https://aws.amazon.com/service-terms/) (GDPR 표준계약 포함) |

## 국외 이전 (처리위탁·보관)

법적 근거: 법 제26조(처리위탁), 제28조의8제1항제3호(계약 이행을 위한 처리위탁·보관).

| 이전받는 자 | 국가 | 이전 항목 | 목적 | 보유기간 | 근거 문서 |
|---|---|---|---|---|---|
| Google LLC | 미국 등 | Google 계정 이메일·식별자·액세스 토큰, 연동 데이터(Sheets·Drive·Classroom), 웹폰트 로딩 시 IP·브라우저 정보 | Google 로그인·연동, 웹폰트 | 연동 해제·계정 파기 시까지 | [Google 개인정보처리방침](https://policies.google.com/privacy) |
| Google LLC — Firebase | 미국 등 | 푸시 기기 토큰, 알림 내용 | FCM 푸시 발송 | 토큰 무효화 시까지 | [Firebase 약관](https://firebase.google.com/terms) |
| Daily, Inc. | 미국 (음성 미디어 서버는 서울 리전) | 이용자 식별자·이름, 음성 스트림, IP | 음성 통화 중계 | 통화 종료 시까지 (저장 안 함) | [Daily 개인정보처리방침](https://www.daily.co/legal/privacy) |
| Cloudflare, Inc. | 미국 등 | IP, 브라우저·기기 정보 | 봇 방지(Turnstile) | 확인 절차 종료 시까지 | [Cloudflare 개인정보처리방침](https://www.cloudflare.com/privacypolicy/) |
| OpenAI, L.L.C. | 미국 | AI 도움 기능 입력 메시지 (회사 서버 경유 — 이용자 IP 미전송) | AI 응답 생성 | 최대 30일 (모델 학습 미사용) | [OpenAI API 데이터 정책](https://openai.com/policies/api-data-usage-policies) |

## 문서 미리보기의 국외 사업자 접근

이용자가 첨부 문서 미리보기를 실행하면 문서 URL이 Microsoft(Office 뷰어) 또는 Google(Docs 뷰어)에 전달되어
해당 사업자가 문서 내용에 접근하고, 이용자의 IP·브라우저 정보가 전송됩니다. 위탁 계약이 아닌 이용자 선택에
의한 접근이므로 처리방침 제8조 ②에 별도 고지했습니다. 미리보기 기능은 소유자 결정에 따라 유지하며, 원하지
않는 이용자는 파일을 내려받아 확인할 수 있습니다.

## 점검 체크리스트 (연 1회)

- 각 수탁자의 약관·DPA 링크가 유효하고 내용 변경이 없는지 확인
- 실제 코드가 전달하는 항목이 표와 일치하는지 확인 (특히 Daily.co 호출부 `src/message`)
- 새 외부 서비스 도입 시: 방침 제7·8조 개정 → 이 표 갱신 → 배포
