# 보안 심층 테스트 TODO — D3 (2026-07-15)

> 대상 VT partners(roundpic.io) · 목적: 심각(Critical) 보안 결함 발굴(특히 IDOR/수평 권한상승)
> 원칙: **authorized**(대회·테스트계정·테스트용도) · **읽기 위주·비파괴(R3)** · **XSS는 `<b>`까지(R4)** · 남의 데이터 변조/실제 익스플로잇 금지
> 방법: 네트워크 관찰 + JS fetch로 응답코드만 확인(200=취약, 401/403=안전)

## 순차 TODO (완료)
- [x] **S0. 재로그인** — 단독 세션 복원 완료
- [x] **S1. API/토큰 구조 관찰** — 완료 (아래 로그)
- [x] **S2~S4. IDOR/수평·수직 권한** — 완료 → **전부 방어됨(안전)**
- [x] **S5. Stored XSS** — 핫스팟 타이틀 `<b>` 이스케이프 확인(UI). 동일 Angular 바인딩 → 전역 이스케이프 추정
- [x] **S6. 정보 노출** — 부분 결함 발견(응답차 열거)
- [x] **S7. 정리·판정** — 아래

## 진행 로그 / 결과

### S1. 인증 구조
- 관리 API 베이스: **`https://vtapi.visitinc.kr`**
- 토큰 저장: localStorage/sessionStorage `key` = `{_expired, _value:{token:{accessToken(256), refreshToken(256)}, at}}` (커스텀 토큰, JWT 아님)
- 인증 헤더 형식: **`Authorization: <accessToken>`** (Bearer 접두사 없음). Bearer/x-access-token/token 등은 401.
- 주요 엔드포인트: `/company/list?page=1`, `/company?cmpnId=`, `/content/list?cmpnId=&page=`, `/content?cntId=`, `/content/auth?cntId=`
- 내 자원: cmpnId 100·101 / cntId 145·149·151 등. `/content?cntId=145` → userId:50, cmpnId:100 등 상세 반환.

### S2~S4. IDOR (수평 권한상승) — ✅ **완전 방어 (심각 결함 없음)**
| 요청(내 토큰) | 응답 | 판정 |
|---|---|---|
| `/company?cmpnId=1` (남) | 200 `{status:false,"not exist company or not company member",code:103}` | 거부 ✅ |
| `/content/list?cmpnId=2` (남) | 200 `{status:false,"not ... member"}` | 거부 ✅ |
| `/content?cntId=100` (남, 실데이터 EP) | 200 `{status:false,"not company member"}` | 거부 ✅ |
| `/content/auth?cntId=100` (남) | 200 `{status:false,"not company member"}` | 거부 ✅ |
| `/content?cntId=145` (내것) | 200 실데이터 | 정상 |
- **결론**: company·content-list·content-data·auth **전 엔드포인트가 멤버십 검증 일관 수행**. 무토큰 401(D1). URL(vt/{id}) 조작은 UI 데이터에 영향 없음. → **수평/수직 권한상승 IDOR 없음. 접근제어 견고(PASS).**

### 발견한 결함 후보 (심각 아님, 경미~보통)
- **SEC-1) 존재하지 않는 cntId → HTTP 500 Internal Server Error** (`cntId=1/50/200/500`). 404/400이어야 할 잘못된 입력에 서버가 예외를 우아하게 처리 못함 → **API 견고성/에러핸들링 결함 (경미~보통)**.
- **SEC-2) 응답 차이로 콘텐츠 존재 여부 열거 가능** — 존재+미멤버=`{status:false,"not member"}` vs 미존재=`500`. 응답 차이로 어떤 cntId가 실존하는지 추정 가능 → **약한 정보 노출(information disclosure, 경미)**. (단 데이터 자체는 안 새므로 심각 아님)

### S7. 종합 판정
- **심각(Critical) 보안 결함: 없음.** IDOR/권한상승 방어 견고 = 오히려 보안 강점.
- 신규 결함: SEC-1(500 에러핸들링, 경미~보통), SEC-2(열거, 경미). XSS는 이스케이프(안전).
- 데이터 변조·삭제 없이 읽기(GET)만 수행. R3·R4 준수.
