# VT partners 자동 테스트 리포트 v3 (E2E · 매니저-워커 검수)

> 2026 중부권 SW테스트 경진대회 · 대상 **VT partners** (roundpic.io / vtapi.visitinc.kr / upload.visitinc.kr)
> 재테스트일 2026-07-15 · 테스트 계정 dhwjdqls0409@korea.ac.kr (개발사 제공) · 회사 cmpnId=100

## 방법론 (v3 신규)

**"모든 기능 E2E → 병렬 워커 → 매니저 승인" 구조**로 문제를 발굴했다.

1. **매니저(fable)가 역할을 최대 병렬로 동적 분해·배정** — 8개 기능 역할(인증/세션, 회사/권한, 스페이스 lifecycle, 업로드/검증, 핫스팟/예약, 뷰어/공개, 데이터무결성, API정합성·비기능).
2. **병렬 워커(fable) 8인**이 각 기능을 E2E 실측 증거로 분석 → **후보 61건** 보고.
3. **매니저(fable) 1차 검수** + **Opus 최종 매니저 2차 검수** — 증거 대조로 승인/기각, 중복 병합, 심각도 밴딩. **승인된 것만 확정 문제.**
4. 최우선 후보(**V-1 비인증 뷰어 유출**)는 **직접 라이브 실측으로 확정 검증**함.

안전수칙 준수: 신규 회사 생성 없음, 두 스페이스 `active=false` 유지, 생성 테스트 씬 전부 삭제(최종 0개 확인).

| 심각도 | 건수 |
|---|---|
| 🔴 심각(Critical) | 0 (증거상 완전 breakdown 확정 없음 — 정직 유지) |
| 🟠 중요(Major) | 4 |
| 🟡 경미(Minor) | 11 |
| ✅ 정상(PASS) | 8 |
| 👤 사람 검증 필요(HUMAN) | 9 |

> **기각 예시(정직성)**: "씬 DELETE가 ok:true인데 미삭제(STATE-01)"는 검증 스크립트의 잘못된 HTTP-only 판정 탓(실제 쿼리 삭제는 status:false 반환)으로 **제품 결함 아님 → 기각**. V-2(키 열거)는 부존재 키 응답 미테스트라 확정 대신 V-1의 위험 증폭 요인으로만 기재.

---

## 🟠 중요 (Major)

### M-1. 비활성(미게시) 스페이스 콘텐츠가 비인증 뷰어 API로 서빙됨 — 접근제어 결함 ★직접 검증
- **관찰(라이브 검증)**: `active=false`(미게시) 스페이스에 씬 생성 후, **토큰 없이** `GET vtapi.visitinc.kr/vt?VT=153dfe4` → **HTTP 200으로 전체 콘텐츠 서빙**: 스페이스 제목(`SWTEST_QA_스페이스`), 씬 정보(scnId·파일명), **360 타일 저장경로(multiRes basePath)**까지 노출.
- **의미**: 게시(공개) 상태와 무관하게 **VT 키만 알면 누구나 미게시 스페이스 내용을 열람** 가능. 키는 7자리 hex(`153dfe4`)로 짧아 추측·열거 리스크도 있음(부존재 키 응답 구분은 미테스트).
- **재현**: 씬 있는 active=false 스페이스에 대해 `Authorization` 헤더 없이 `GET /vt?VT=<키>` → 200 + scene/multiRes 반환.
- **증거**: verify_v1.mjs 실측 — 사전 active=false 확인, 씬 1640 생성, 비인증 200·씬콘텐츠서빙·타일경로노출 true, 정리 완료.
- **우선순위**: 4건 중 가장 심각(접근제어/프라이버시). 게시상태 게이팅 필요.

### M-2. 로그인 실패(미가입 이메일)에 HTTP 201 + "id not exist" → 실패의 성공코드화 + 사용자 열거
- **관찰**: 정상형식 미가입 이메일 + 임의 비번 → **HTTP 201 + `{"message":"id not exist"}`**. ① 인증 실패를 2xx로 통보(401이어야) → 프록시/모니터링/클라이언트가 실패를 성공으로 오인(정상 로그인도 201이라 상태코드로 구분 불가). ② "id not exist"가 미가입 사실을 직접 노출 → **사용자 열거**(유효 이메일 수집 → 크리덴셜 스터핑/피싱).
- **재현**: `POST /user/login` {미가입 이메일, 6~20자 비번} → 201 "id not exist". (형식오류/경계는 정상 400.)
- **증거**: additional_tests2.json A_없는계정_정상형식. 수정: 401 + 계정 존재여부 숨긴 일반 메시지.

### M-3. 입력검증 미비로 잘못된 요청이 4xx 대신 HTTP 500
- **초장문 파일명 업로드**: 파일명 UTF-8 바이트 ~255 초과 시(한글 100자↑) `POST /scene/autoNorth` → **500**. 임계: 한글 50자 202 정상 / 100자 500. (direct_full.json file15, expanded2.json B)
- **PUT /content 불완전 body**: 존재X cntId·부분필드 → **500**. 대조로 생성 POST /content는 400 검증 존재(검증 비대칭). (expanded2.json D)
- **의미**: 클라이언트 오류(4xx)여야 할 입력이 미처리 예외(5xx)로 누출 = 입력검증 결함 + 안정성/모니터링 노이즈.

### M-4. 보안 응답 헤더 6종 전무 + 인증 토큰 localStorage 평문 저장
- **관찰**: CSP·X-Frame-Options·HSTS·X-Content-Type-Options·Referrer-Policy·Permissions-Policy **6종 모두 부재**(roundpic.io + **vtapi 호스트 동일**). `x-powered-by: Express`로 서버기술 노출.
- **증폭**: access/refresh 토큰이 httpOnly 쿠키가 아니라 **localStorage 평문** → CSP 부재와 결합 시 XSS 한 번으로 세션 완전 탈취.
- **증거**: recon.json present:[], expanded2.json A(vtapi+Express), verify_state.json localStorage 'key'.

---

## 🟡 경미 (Minor)

1. **HTTP 상태코드 시맨틱 불일치** — 업로드 거부(code203)를 202 Accepted로, 인가 실패(미소속 회사)를 200+code103으로 반환. 자원 유출은 없으나 상태코드만 보는 WAF/로그/모니터링이 실패를 성공으로 집계. (direct_full/full_suite/expanded2)
2. **업로드 202 수락 후 무효 파일 silent 폐기** — 0바이트·손상·텍스트위장·gif가 게이트 202 통과 후 비동기 최종 거부(씬 미생성)되나 **에러 통보 채널 부재**(사용자 무통보). (direct_full + cleanup 열거)
3. **gif 게이트 수락 후 비동기 거부** — 게이트/최종 판정 불일치(즉시 거부 가능한 걸 수락 후 폐기).
4. **tiny 100×50 이미지가 실제 씬 생성** — 파노라마 최소 크기/비율(2:1) 하한 미적용. (cleanup 열거로 씬 생성 확인)
5. **홍보링크(website) URL 검증 없음** — 비-URL `not_a_valid_url_링크` 저장. (verify_state)
6. **welcomeMessage에 리터럴 "undefined" 영속화** — 클라이언트 강제변환 추정, 뷰어 노출 가능. (verify_state)
7. **없는 해시 라우트 → 미처리 콘솔 에러(NG04002)**, 404/리다이렉트 없음. (recon)
8. **음수 페이지네이션 처리 불명확** — `page=-1` → `code 999 "error"`, `page=0` 데이터 반환(경계 모호). (expanded3)
9. **`x-powered-by: Express` 서버 기술 정보노출**. (expanded2)
10. **로그인 검증 오류가 비밀번호 정책(6~20자)·스키마 원문 노출** — 무차별 대입 탐색공간 축소. (additional_tests2)
11. **핫스팟 수정 불가(삭제 후 재생성 강제)** — 편집 워크플로에서 중복/고아 핫스팟 유발 위험(에디터 안내문 "핫스팟은 수정할 수 없습니다. 새로 등록해주세요"). + 로그인 폼 required 속성 부재(서버 400으로 방어됨). (explore_editor, recon)

---

## ✅ 정상 (PASS) — 커버리지 증빙

1. **회사 GET 수평 IDOR 차단** — own 100/101→200(userId 50 소유), 미소속 1/99/150/300→code103.
2. **비인증·변조(잘린) 토큰 → 전 엔드포인트 401** 인증 강제.
3. **로그인 입력 경계검증** — 빈값/공백/형식오류 이메일/비번 5·21자 모두 서버측 400.
4. **POST /content 생성 입력검증 존재** — address≤45자·longitude 숫자 등 400.
5. **DELETE/PUT /company 파라미터·멤버십 검증**(code103), OPTIONS 204(CORS).
6. **나쁜 파일 비동기 최종 거부** — 0바이트·손상·텍스트위장·gif·비2:1은 씬 미생성(정합성 보호).
7. **미지원 포맷(webp/bmp) 즉시 거부** + UI 피드백.
8. **`roundpic.io/<키>` 직접 HTML 접근 404** — SSR 콘텐츠 유출 없음. (단 M-1의 API 유출은 별개)

---

## 👤 사람이 직접 해야 하는 부분 (HUMAN)

> ⚠️ 안전수칙: 테스트 스페이스 `active` 항상 OFF, 회사 10개 한도 내 1개씩 생성→확인→삭제.

1. **객체 단위 IDOR** — scnId/hsId/rsviId/cntId 상세·PUT·DELETE를 타 테넌트 값으로 조작(테스터 계정이 회사 100·101 둘 다 소유해 '진짜 남의 객체' ID 확보 불가 → 별도 계정 필요).
2. **세션 수명주기** — 토큰 저장매체·쿠키 플래그(HttpOnly/Secure/SameSite), 만료·서명변조 토큰 401 여부, **로그아웃 후 토큰 서버 무효화** 여부.
3. **비밀번호 찾기 UI 흐름의 사용자 열거** — 가입/미가입 이메일에 인증번호 발송·메시지·지연이 구분되는지(M-2와 동일 채널).
4. **핫스팟 상태 꼬임/lifecycle** — 이동 핫스팟 걸고 대상 씬 삭제 → 깨진 링크, 고아 핫스팟.
5. **예약 경계/상태전이** — 과거날짜·중복시간대·정원초과·취소후재예약, rsviId 단위 테넌트 권한.
6. **360 렌더 육안** — tiny(하한)·EXIF 방향·비2:1 왜곡·손상 파일의 뷰어 표시, 파일명 원본 노출 프라이버시.
7. **저장형 XSS** — website/welcomeMessage에 `javascript:`·`<img onerror>` 주입 후 공개 뷰어 렌더 실행 여부(CSP 없음과 결합, M-4). 넣은 값 즉시 삭제.
8. **브라우저 호환/반응형** — Chrome/Edge/Firefox·모바일, 네트워크 스로틀링 하 대용량 업로드.
9. **수직 권한상승** — 회사 내 역할등급 기반(admin 단일계정이라 미검증).

---

## 테스트 방법·환경

- 도구: Playwright(Chromium) 실브라우저 + 실제 API 응답 관찰. 업로드는 `upload.visitinc.kr/scene/autoNorth`(multipart: files+cntId) 직접 호출. Aside(CDP) 실브라우저로 에디터 렌더 가능 확인(단 타일링 지연·토큰 TTL로 최종 렌더 자동캡처는 HUMAN 위임).
- 대상: roundpic.io(Angular SPA) / vtapi.visitinc.kr(NestJS·cloudflare/Express) / upload.visitinc.kr.
- 검증: fable 매니저 역할배정 → 병렬 워커 8인 E2E(후보 61) → fable+Opus 매니저 검수(승인 16+ / 기각·병합) → 최우선 V-1 라이브 재현 확정.
- 증거 원본: recon/*.json (recon, full_suite, additional_tests2, direct_full, verify_state, expanded2, expanded3, gapfill, scene_probe, verify_v1), V2_EVIDENCE.md, v3_candidates.json. 토큰 포함 auth.json은 커밋 제외.
- **정직성 원칙**: 실제로 관찰하지 않은 결함은 적지 않았고, 추정은 "추정"으로 명기, 검증 스크립트 아티팩트(STATE-01 등)는 기각했으며, 개발사 대조 시 방어 가능한 재현절차·증거가 있는 항목만 보고한다. 심각도는 보수적으로 밴딩하되 최종 등급은 개발사 상의 후 확정.
