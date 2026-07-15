# VT partners 자동 테스트 리포트 v2

> 2026 중부권 SW테스트 경진대회 · 대상 **VT partners** (roundpic.io / vtapi.visitinc.kr / upload.visitinc.kr)
> 재테스트일 2026-07-15 (개발사 앱 재배포 후, vendor 41e4beb) · 테스트 계정 dhwjdqls0409@korea.ac.kr (개발사 제공) · 회사 cmpnId=100

## v1 대비 개선점

- **처음부터 재실측** + 업로드는 에디터 UI 대신 **API 직접 호출**로 안정 재현(신 빌드에서 헤드리스 에디터 캔버스 미초기화 우회).
- **비동기 최종결과까지 확정** — v1의 "게이트 통과, 최종 미확정" 헷지 해소.
- **Opus 에이전트 68개**로 교차검증: 후보 62건 → 적대적 검증(refute-by-default) 통과 45건 → 심각도별 합성. 과장·오분류·추정 필터링.
- 안전수칙 준수: 신규 회사 생성 없음, 두 스페이스 `active=false` 유지, 생성 테스트 씬 **전부 삭제(최종 0개 확인)**.

| 심각도 | 건수 |
|---|---|
| 🔴 심각(Critical) | 0 (증거상 breakdown 확정 없음 — 정직 유지) |
| 🟠 중요(Major) | 3 |
| 🟡 경미(Minor) | 9 |
| ✅ 정상(PASS) | 7 |
| 👤 사람 검증 필요(HUMAN) | 6 |

> **심각도 밴딩 주의**: 검증 에이전트는 개발사 반박 방어를 위해 대부분을 보수적으로 경미로 판정했다. 아래 "중요 3건"은 웹 보안 표준(OWASP/일반 pentest)상 통상 Medium으로 평가되는 항목을 정직하게 승격한 것으로, **최종 등급은 개발사 상의 후 확정**된다.

---

## 🟠 중요 (Major)

### M-1. 초장문 파일명 업로드 시 서버 500 (미처리 예외)
- **관찰**: 정상 이미지라도 파일명이 과도하게 길면 `POST upload.visitinc.kr/scene/autoNorth` → **500 Internal Server Error**. 클라이언트 입력 오류는 4xx(400/413/422)여야 하는데 500은 서버 결함 노출.
- **임계**: 한글 50자 → 202 정상, **한글 100자 → 500** (UTF-8 바이트 ~255 초과 추정, DB 컬럼/경로 길이 한계). 파일명 길이 상한 검증 부재.
- **재현**: 파일명 100자↑ 한글 유효 이미지를 multipart(cntId 포함)로 업로드 → 500.
- **증거**: direct_full.json file15, expanded2.json B(임계), 재배포 새 빌드에서도 재현.

### M-2. 로그인 실패에 HTTP 201 + 계정 존재여부 노출 (사용자 열거)
- **관찰**: 미가입 정상형식 이메일 + 임의 비번 → 인증 실패인데 **HTTP 201 Created + `{"message":"id not exist"}`**.
  - ① 로그인은 리소스 생성이 아니므로 201 부적절(정상은 401) → 2xx를 성공으로 보는 자동화 클라이언트가 실패를 성공으로 오인 가능.
  - ② "계정 없음"을 명시해 가입/미가입 이메일을 구분 노출 → **사용자 열거**로 유효 이메일 수집 후 크리덴셜 스터핑·피싱 악용 가능. 인증 실패는 계정 존재 여부와 무관한 동일 메시지여야 함.
- **재현**: `POST vtapi.visitinc.kr/user/login` 에 미가입 이메일 + 6~20자 비번 → 201 "id not exist".
- **증거**: additional_tests2.json A_없는계정_정상형식 (대조: 형식오류/빈값/경계는 모두 400).

### M-3. 보안 응답 헤더 6종 전무 + 인증 토큰 localStorage 평문 저장
- **관찰**: CSP·X-Frame-Options·HSTS·X-Content-Type-Options·Referrer-Policy·Permissions-Policy **6종 모두 부재**(웹 roundpic.io + **API vtapi 호스트 동일**). 추가로 응답에 `x-powered-by: Express`(서버 기술 노출).
- **증폭 요인**: 인증 access/refresh 토큰이 httpOnly 쿠키가 아니라 **localStorage에 평문 저장** → CSP 부재와 결합 시 **XSS 한 번으로 세션 완전 탈취**로 직결.
- **재현**: 비인증으로 roundpic.io/vtapi 응답 헤더 확인(6종 부재) + 로그인 후 `localStorage.getItem('key')`에 accessToken/refreshToken 평문.
- **증거**: recon.json 보안헤더 present:[], expanded2.json A(vtapi 동일+Express), verify_state.json localStorage 'key'.

---

## 🟡 경미 (Minor)

### m-1. `PUT /content`(스페이스 수정)에 불완전 body → 500
불완전/부분 필드 업데이트 요청에 400(검증 오류) 대신 500 미처리 예외. (수정 엔드포인트는 PUT /content가 유일 — PATCH·POST update·PUT /content/:id 는 404) — expanded2.json D.

### m-2. 업로드 무효 파일을 202 accepted로 수락 후 비동기 조용히 폐기 (판정 시점 불일치·무통보)
동기 게이트가 사실상 확장자 화이트리스트(webp·bmp만 차단)만 수행 → 0바이트·손상·텍스트위장(.jpg)·gif·비2:1·초소형을 `accepted:true`(202)로 수락. 이후 비동기에서 다수가 씬 미생성으로 **조용히 폐기(사용자 피드백 없음)**. 즉시 거부 가능한 입력을 수락 후 폐기하는 시점 불일치 — direct_full.json + 비동기 최종결과.

### m-3. 업로드 거부(code203)를 HTTP 202 Accepted로 반환 (상태코드 시맨틱 불일치)
미지원 포맷(webp/bmp)을 즉시 거부(status:false, code203)하면서 HTTP는 202. 거부는 415 등 4xx가 적절. 상태코드만 보는 모니터링은 실패를 성공으로 오인 가능(UI는 body 파싱으로 완화) — direct_full.json 07/09.

### m-4. 초소형 이미지(100×50) 씬 생성 — 크기/비율 하한 없음
tiny 100×50이 최종 씬으로 생성됨. 파노라마(2:1 대형) 도구인데 하한 검증 없음. 뷰어 렌더 깨짐은 HUMAN 확인 — direct_full/cleanup 열거.

### m-5. 홍보링크(website) URL 형식 검증 없음
URL이어야 할 website에 한글 포함 비-URL 문자열(`not_a_valid_url_링크`)이 검증 없이 저장/반환. 로그인 이메일·비번은 엄격 검증되는 것과 비대칭. 뷰어 렌더 시 저장형 XSS 벡터 가능성(HUMAN) — verify_state.json.

### m-6. welcomeMessage에 리터럴 "undefined" 영속화
미입력 시 JS undefined가 문자열 "undefined"로 강제변환 저장(클라이언트 직렬화 버그 추정). 뷰어 노출 가능. 두 스페이스 공통 — verify_state.json.

### m-7. 없는 해시 라우트 → 미처리 콘솔 에러(NG04002), 404/리다이렉트 없음
라우터 매칭 실패를 404/리다이렉트로 처리 못 하고 uncaught promise 예외(NG04002), 무관한 비밀번호찾기 화면으로 폴백. 번들 스택트레이스 콘솔 노출 — recon.json.

### m-8. 로그인 검증 오류 메시지가 내부 비밀번호 정책(6~20자)·타입 규칙 노출
class-validator 원문("password must be longer/shorter than or equal to 6/20", "must be a string") 그대로 반환 → 무차별 대입 탐색공간 축소. 상태코드 400은 정확 — additional_tests2.json.

### m-9. 음수 페이지네이션 처리 불명확
`page=-1` → status:false `code 999 "error"`(일반화된 불명확 에러). `page=0`은 데이터 반환(경계 모호), `page=abc`는 400(정상). 음수 입력 처리 일관성 부족 — expanded3.json A.

---

## ✅ 정상 (PASS) — 커버리지 증빙

- **권한/IDOR 정상**: 소유 회사(100·101, userId 50) 200, 미소속(1·99·150·300) code103 차단, 비인증 401.
- **로그인 서버측 검증**: 빈값/형식오류/비번 경계(5·21자) 모두 400(class-validator).
- **나쁜 파일 비동기 최종 거부**: 0바이트·손상·텍스트위장·gif·비2:1은 결국 씬 미생성(정합성 유지).
- **미지원 포맷 즉시 거부 + UI 피드백**: webp/bmp code203 + "업로드 요청에 실패했습니다".
- **한글·공백·특수문자 파일명** 정상 씬 생성.
- **`GET /user`**는 인증된 본인 프로필 반환(취약점 아님).
- **HTTP 메서드 처리**: OPTIONS 204(CORS), DELETE/PUT /company code103 정상 처리.

---

## 👤 사람이 직접 해야 하는 부분 (자동/시각 판단 불가)

> ⚠️ 안전수칙: 테스트 스페이스 `active` 항상 OFF, 회사 10개 한도 내 1개씩 생성→확인→삭제.

1. **360 뷰어 렌더·EXIF 방향·핫스팟** — 수락된 이미지(초대형·초소형·png·EXIF 회전)가 뷰어에서 올바로 렌더/방향/핫스팟 배치되는지 육안 확인.
2. **저장형 XSS** — welcomeMessage/website 등에 `javascript:`·`<img onerror>` 주입 후 공개 뷰어/공유 페이지 렌더 시 실행되는지(CSP 없음과 결합). 넣은 값 즉시 삭제.
3. **업로드 202 후 UI 실패 표시 + jobId 작업상태 폴링 API** 유무 — 있으면 "무통보 폐기(m-2)"가 아닐 수 있어 실제 에디터에서 확인.
4. **세션 수명주기** — 로그아웃 후 서버측 토큰 무효화 여부, 만료·refresh 회전(탈취 토큰 재사용 위험, M-3과 연계).
5. **핫스팟 상태 꼬임 / 삭제 lifecycle / 예약 핫스팟** — 이동 핫스팟 걸고 대상 씬 삭제(깨진 링크), 삭제한 VT 목록 잔존, 예약 과거날짜/경계/동시.
6. **새 빌드 에디터 UI 흐름 + 브라우저 호환** — 헤드리스에서 캔버스 미초기화로 UI 진행률/오류표시 자동 검증 불가. 실브라우저에서 사람이 확인, Chrome/Edge/Firefox·모바일 반응형 포함.

---

## 테스트 방법·환경

- 도구: Playwright(Chromium) 실브라우저 + 실제 API 응답 관찰. 업로드는 `upload.visitinc.kr/scene/autoNorth`(multipart: files+cntId) 직접 호출.
- 대상: roundpic.io(Angular SPA) / vtapi.visitinc.kr(NestJS 추정, cloudflare/Express) / upload.visitinc.kr.
- 검증: Opus 에이전트 68개 — 5개 렌즈 독립분석 → 각 결함 적대적 검증(refute-by-default) → 심각도별 합성. 후보 62 → 통과 45 → 병합.
- 증거 원본: recon/*.json (recon, full_suite, additional_tests2, direct_full, verify_state, expanded2, expanded3), V2_EVIDENCE.md. 토큰 포함 auth.json은 커밋 제외.
- **정직성 원칙**: 실제로 관찰하지 않은 결함은 적지 않았고, 추정은 "추정"으로 명기했으며, 개발사 대조 시 방어 가능한 재현절차·증거가 있는 항목만 보고한다.
