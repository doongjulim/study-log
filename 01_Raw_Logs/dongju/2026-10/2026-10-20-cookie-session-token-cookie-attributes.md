---
date: 2026-10-20
type: raw
author: dongju
tags: [쿠키, cookie, 세션, session, 토큰, token, JWT, json-web-token, 무상태성, stateless, stateful, Set-Cookie, Cookie헤더, Authorization, Bearer, localStorage, 로컬스토리지, 서명, signature, HMAC, Base64Url, 인코딩, encoding, 암호화, encryption, 해시, hash, bcrypt, 느린해시, slow-hash, 오프라인대입공격, offline-brute-force, rate-limiting, HttpOnly, Secure, SameSite, Strict, Lax, None, CSRF, 사이트간요청위조, XSS, 크로스사이트스크립팅, 탈취vs악용, theft-vs-abuse, CSP, 출력이스케이프, HTTPS, TLS, 쿠키보안속성, cookie-attributes, 면접답변]
topic: "쿠키는 저장·전송 수단이고 세션/토큰은 인증 상태를 어디에 두느냐(서버 vs 클라이언트)의 차이다. HttpOnly 는 XSS 의 쿠키 탈취를, Secure 는 평문 HTTP 노출을, SameSite 는 CSRF 를 막는다"
summary: "HTTP 는 무상태라 요청 사이를 기억하지 못하고, 매 요청 비밀번호 검증은 일부러 느린 해시(bcrypt)를 매번 돌리는 일이라 비싸다. 그래서 로그인 때 한 번만 검증하고 결과를 세션 ID 나 토큰으로 들고 다닌다. 쿠키는 그 값을 담아 자동 전송하는 수단이라 세션·토큰과 같은 층위가 아니고, 세션 vs 토큰의 진짜 축은 상태를 서버가 갖느냐(stateful) 클라이언트가 갖느냐(stateless)다. JWT 서명은 위조를 막지만 payload 는 Base64Url 인코딩이라 누구나 읽는다. HttpOnly 는 JS 의 쿠키 읽기(탈취)만 막고 페이지 안 요청(악용)은 못 막으며, SameSite 는 다른 사이트발 요청의 쿠키를 제한해 CSRF 를 막는다(Lax 는 최상위 GET 이동만 허용)."
source: session
distilled: true
distilled_at: 2026-10-22
---

## 배운 개념

### 1. 왜 필요한가 — 무상태 HTTP + 비싼 검증

- HTTP 는 **무상태(Stateless)**: 서버는 1번 요청과 2번 요청이 같은 사람에게서 왔는지 프로토콜 차원에서 모른다.
- 상태 유지 장치가 없으면 매 요청마다 아이디·비밀번호를 보내야 함 → ① 민감정보가 매번 오가며 노출 기회 증가, ② 서버는 매번 **비밀번호 해시 검증**을 반복.
- 비밀번호 해시(bcrypt)는 **일부러 느리게**(1회 수십~수백 ms) 설계돼 있어 매 요청 반복은 감당 불가.
- → 로그인 때 **딱 한 번** 비싼 검증을 하고, 그 결과를 **세션 ID 나 토큰**으로 바꿔 들고 다닌다.

### 2. 층위 구분 — 쿠키는 비교 대상이 아니다

| 개념 | 정체 |
|---|---|
| 쿠키 | 브라우저의 **저장 + 자동 전송 수단** |
| 세션 | 인증 상태를 **서버**에 두는 방식 (Stateful) |
| 토큰 | 인증 상태를 **토큰 안에 담아 클라이언트**가 들고 다니는 방식 (Stateless) |

- 세션 방식도 브라우저는 **세션 ID** 를 보내야 서버가 세션을 찾는다 → 보통 쿠키에 담는다.
- 토큰도 쿠키에 담을 수 있다 → "쿠키 vs 토큰"은 어색한 비교. 진짜 축은 **상태를 어디에 보관하느냐**.

### 3. 쿠키 vs localStorage — 전송 방식이 다르다

```
서버 → 브라우저:  Set-Cookie: SESSION=abc; HttpOnly; Secure; SameSite=Lax
브라우저 → 서버:  Cookie: SESSION=abc          ← 해당 도메인 요청마다 '자동'

localStorage:    JS 가 직접 꺼내서
                 Authorization: Bearer <token>  ← '수동'
```

| 저장소 | 특징 | 취약한 공격 |
|---|---|---|
| 쿠키 | 자동 전송 | **CSRF** (내 권한으로 원치 않는 요청이 나감) |
| localStorage | JS 로 읽기 가능 | **XSS** (심어진 스크립트가 토큰을 읽어 감) |

### 4. JWT — 위조는 막고, 열람은 못 막는다

```
header.payload.signature
signature = HMAC( base64url(header) + "." + base64url(payload), 서버만 아는 secret )
```

- payload 를 `role: ADMIN` 으로 바꾸면 시그니처가 완전히 달라지는데, 공격자에겐 secret 이 없어 새 시그니처를 만들 수 없다. 서버는 받은 header·payload 로 **다시 계산**해 붙어 온 값과 비교.
- payload 는 **Base64Url 인코딩**일 뿐 → 키 없이 누구나 디코드(jwt.io).
- 넣어도 되는 것: 식별자(`sub` 같은 사용자 ID — 없으면 토큰이 기능을 못 함), 권한, 만료. 넣으면 안 되는 것: 비밀번호, 주민번호, 연락처 등 **민감정보**. 기준은 "남에게 보여도 되는가".

### 5. 인코딩 · 암호화 · 해시

| | 되돌릴 수 있나 | 키 필요 | 용도 |
|---|---|---|---|
| 인코딩 | ⭕ 누구나 | ❌ | 형식 변환 (JWT payload) |
| 암호화 | ⭕ 키가 있으면 | ⭕ | 기밀성 |
| 해시 | ❌ 단방향 | ❌ | 비밀번호 저장, 무결성 |

- 비밀번호를 암호화로 저장하면 DB 와 키가 함께 털렸을 때 **전부 복호화**된다 → 해시를 써야 한다.
- bcrypt 가 느린 이유: 유출된 DB 로 하는 **오프라인 대입 공격**에는 Rate Limiting 이 작동하지 않는다. 해시 1회가 느리면 수십억 후보 대입이 현실적으로 불가능해진다.

### 6. 쿠키 보안 속성

| 속성 | 동작 | 막는 것 | 막지 못하는 것 |
|---|---|---|---|
| **HttpOnly** | JS 의 `document.cookie` 접근 차단 | XSS 로 쿠키 **탈취** | XSS 스크립트가 페이지 안에서 `fetch` 하는 **악용** (쿠키는 브라우저가 자동으로 실음) |
| **Secure** | **HTTPS** 연결에서만 전송 | 평문 HTTP 스니핑 (공용 와이파이) | — |
| **SameSite** | 다른 사이트에서 시작된 요청의 쿠키 제한 | **CSRF** | 같은 사이트 안의 공격 |

SameSite 값별 — (가) 악성 사이트의 링크 클릭(최상위 GET 이동), (나) 악성 사이트의 숨겨진 `<form method="POST">`:

| 값 | (가) | (나) | 비고 |
|---|---|---|---|
| Strict | ❌ 안 실림 | ❌ 안 실림 | 메일·검색 링크로 들어오면 로그아웃 상태로 보이는 UX 불편 |
| Lax (기본값) | ⭕ 실림 | ❌ 안 실림 | 상태를 바꾸는 cross-site POST 만 막고 링크 이동은 살림 |
| None | ⭕ | ⭕ | `Secure` 필수 |

- ⚠️ Lax 의 전제는 "GET 은 읽기 전용". `GET /transfer?to=...` 같은 API 가 있으면 Lax 로도 못 막는다.
- XSS 자체의 근본 대응은 **출력 이스케이프 + CSP**. HttpOnly 는 피해 축소 장치.

### 면접 답변

> "HTTP 는 무상태라 로그인 상태를 따로 유지해야 합니다. 쿠키는 그 값을 브라우저에 저장하고 자동으로 보내는 수단이고, 세션과 토큰은 인증 상태를 서버에 두느냐 클라이언트가 들고 다니느냐의 차이입니다. 그래서 토큰도 쿠키에 담을 수 있습니다. 쿠키의 보안 속성 중 HttpOnly 는 JS 가 쿠키를 읽지 못하게 해서 XSS 로 쿠키가 탈취되는 걸 막지만, 스크립트가 페이지 안에서 요청을 보내는 것까지 막지는 못합니다. Secure 는 HTTPS 에서만 쿠키를 보내 평문 노출을 막고, SameSite 는 다른 사이트에서 시작된 요청에 쿠키를 싣지 않아 CSRF 를 막습니다. 기본값인 Lax 는 링크 이동 같은 GET 은 허용하고 cross-site POST 만 막는데, 그래서 GET 으로 상태를 바꾸는 API 를 만들면 안 됩니다."

## 오답·헷갈린 점

- **전**: 서버가 기억 못 하는 이유를 "요청된 정보에만 응답한다"로 설명, 용어 기억 못 함 → **후**: **무상태성(Stateless)**. 요청과 요청 사이를 연결해 기억하지 않는다.
- **전**: "리소스를 잡아먹는다" → **후**: 구체적으로 **일부러 느린 비밀번호 해시**를 매 요청 반복하는 비용.
- **전**: 비밀번호를 "**암호화**해서 비교" → **후**: **해시**. 핵심 차이는 수단(키/수식)이 아니라 **되돌릴 수 있느냐**.
- **전**: bcrypt 가 느린 이유 = "짧은 시간 반복 요청을 해킹으로 감지" → **후**: 그건 **Rate Limiting**. 느린 해시는 Rate Limiting 이 안 통하는 **오프라인 대입 공격**의 비용을 올리려는 것.
- **전**: 세션 ID 를 "쿠키나 storage 에" 저장한다고 동등하게 나열 → **후**: 쿠키는 `Cookie` 헤더로 **자동** 전송(CSRF 위험), localStorage 는 JS 가 **수동**으로 `Authorization` 에 담음(XSS 위험).
- **전**: JWT 위조 방지 = "시그니처 값을 비교" → **후**: `HMAC(header.payload, secret)` 을 서버가 **재계산**해 비교. 공격자는 secret 이 없어 새 서명을 못 만든다.
- **전**: JWT payload 는 **암호화**되어 있다 → **후**: **Base64Url 인코딩**. 누구나 읽는다. 서명은 위조만 막는다.
- **전**: payload 에 "식별 가능한 값을 넣으면 안 된다" → **후**: 식별자(`sub`)는 **필요**. 빼야 하는 건 **민감정보**.
- **전**: "HttpOnly → XSS 가 막힌다" (그러면서 "완전히 안전하진 않다") → **후**: HttpOnly 는 **쿠키 탈취**만 막는다. 페이지 안에서 요청을 보내는 **악용**은 못 막는다. (스스로 탈취 vs 악용으로 정리 👍)
- **전**: "Strict 는 허용된다" — Strict/Lax 뒤바뀜 → **후**: **Strict 가 가장 엄격**(링크 이동도 차단). Lax 는 최상위 GET 이동만 허용하는 절충이라 기본값.
- **전**: Secure = "쿠키가 노출된다", "지정된 안전장치가 모두 마련된 경우에만 전송" → **후**: 노출은 Secure **없을 때**의 문제. Secure 는 **HTTPS 에서만** 전송.
- ⚠️ 반복 테마: 후반부로 갈수록 답이 **한 줄로 짧아지고 모호한 표현**("안전장치", "막힌다")으로 끝남. 보안 속성 질문엔 "**무엇을** 막고 **무엇은** 못 막는가"를 짝으로 답할 것. 이전 세션들의 "부정확한 인과 표현" 테마와 같은 계열.

## Q&A

- **Q. 쿠키·세션·토큰이 왜 필요한가?** A. HTTP 가 무상태라서. 매 요청 비밀번호 검증은 느린 해시를 반복하는 일이라 로그인 때 한 번만 하고 결과를 들고 다닌다.
- **Q. 셋의 차이는?** A. 쿠키는 저장·전송 수단, 세션·토큰은 인증 상태 관리 방식. 세션은 서버가 상태를 갖고(stateful), 토큰은 클라이언트가 들고 다닌다(stateless).
- **Q. 사용자가 JWT payload 를 고치면?** A. 서버가 secret 으로 서명을 재계산해 불일치 → 거부. 공격자는 secret 이 없어 새 서명을 못 만든다.
- **Q. JWT payload 는 암호화되나?** A. 아니다, Base64Url 인코딩. 민감정보를 넣지 않는다.
- **Q. 비밀번호를 왜 해시로 저장하나?** A. 단방향이라 DB(와 키)가 털려도 원문 복구 불가. bcrypt 는 일부러 느려 오프라인 대입을 비현실적으로 만든다.
- **Q. HttpOnly / Secure / SameSite 는 각각 무엇을 막나?** A. HttpOnly: XSS 로 쿠키 탈취(악용은 못 막음). Secure: 평문 HTTP 전송 노출. SameSite: CSRF.
- **Q. HttpOnly 면 XSS 에서 안전한가?** A. 아니다. 스크립트가 페이지 안에서 요청을 보내면 쿠키는 자동으로 실린다. XSS 는 출력 이스케이프·CSP 로 막아야 한다.
- **Q. 왜 Lax 가 기본값인가?** A. 상태를 바꾸는 cross-site POST 는 막고, 링크 이동(GET)은 허용해 UX 를 살린다. 단 GET 이 읽기 전용이어야 한다.

## 다음 학습

- CSRF 토큰(Synchronizer Token / Double Submit Cookie)과 SameSite 의 관계 — Spring Security 의 CSRF 설정을 JWT 쿠키 방식에서 끄면 안 되는 경우
- SameSite 의 "site" 정의(eTLD+1)와 origin 의 차이, 서브도메인 간 요청
- CSP 기본 디렉티브와 XSS 방어, React/Thymeleaf 의 기본 이스케이프
- 세션 방식의 수평 확장 문제(스티키 세션 / Redis 세션) vs 토큰 방식의 즉시 무효화 문제(블랙리스트, 짧은 만료 + refresh)
- bcrypt cost factor, salt 의 역할(레인보우 테이블), Argon2
- 연결: [[token-storage-httponly-cookie]], [[refresh-token-rotation]], [[secret-hashing-algorithm-choice]], [[filter-interceptor-aop]]
