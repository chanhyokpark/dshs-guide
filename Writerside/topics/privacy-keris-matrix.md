# 필수기준 대응표

「학습지원 소프트웨어 필수기준 준수를 위한 공교육 에듀테크 개인정보보호 가이드」(KERIS, 2026)의 5대 필수기준과
DSHS.APP 구현의 대응표입니다. 학교 제출용 근거 자료입니다.

| # | 필수기준 | 구현 | 확인 방법 |
|---|---|---|---|
| 1 | 최소한의 개인정보 수집 | 학교 제공 계정 정보와 서비스 제공에 필요한 생성 정보만 수집. 수집 항목 전수는 [](privacy-data-inventory.md) 참조. 미사용 항목은 수집하지 않음 | 처리방침 §2, 데이터 인벤토리 |
| 2 | 개인정보 처리방침 공개 | [dshs.app/privacy](https://dshs.app/privacy)에 15개 조항(2026 개인정보처리방침 작성지침 준수) 공개, 버전 관리 및 이전 버전 열람 제공. 이용약관은 [dshs.app/terms](https://dshs.app/terms) | 로그인 화면·사이드바·설정에서 접근 가능 |
| 3 | 동의 절차 (해당 시) | 학부모 등록 시 개인정보 수집·이용/이용약관 필수 동의 체크박스, 동의 없이는 등록 불가(서버 검증). 동의 버전·일시·IP를 `user_consents`에 기록 | 등록 마법사, DB 동의 기록 |
| 4 | 보유기간·파기 및 안전성 확보조치 | 보유기간 상수화([](privacy-retention.md)) + 주간/일간 자동 파기 크론. 접속기록(감사 기록) 2년 보관·월별 점검([](privacy-management-plan.md)), 웹서버 접속 로그 3개월 보관. TLS·비밀번호 일방향 암호화·역할 기반 접근 통제 | 크론 실행 기록, `/admin/audit` |
| 5 | 정보주체 권리 보장 절차 | 문의 페이지(/contact)의 개인정보 보호책임자 연락 채널로 열람·정정·삭제·처리정지를 요청하면 10일 이내 회신(처리방침 §11). 본인 정보는 설정 > 프로필에서 직접 조회 가능. 삭제 승인 시 관리자가 파기를 예약하고 자동 파기 크론이 집행 | `/contact`, 처리방침 §11·§13 |

## 관련 페이지·코드 위치

- 처리방침·약관: `apps/web/src/lib/privacy/` (버전 레지스트리 `versions.ts`)
- 동의 기록: `apps/backend/src/parent/parent.service.ts` (`register`), `user_consents` 테이블
- 감사 기록: `apps/backend/src/audit/`, `/admin/audit`
- 접속 로그: `apps/backend/src/access-log/`, `apps/web/src/lib/server/access-log.ts`
- 파기: `apps/backend/src/privacy/privacy-purge.service.ts`, k8s `privacy-purge-*` 크론
- 권리 행사 접수 채널: `/contact` (개인정보 보호책임자 이메일)
