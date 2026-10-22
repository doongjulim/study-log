---
type: refined
slug: session-vs-token-auth
tags: [세션, session, 토큰, token, 쿠키, cookie, 세션vs토큰, session-vs-token, 인증상태, authentication-state, 무상태, stateless, 무상태성, 상태유지, stateful, HTTP무상태, http-stateless, 세션ID, session-id, JSESSIONID, Set-Cookie, Cookie헤더, Authorization, Bearer, localStorage, 로컬스토리지, 자동전송, 수동전송, JWT, 로그인, login, bcrypt, 비밀번호검증, 수평확장, scale-out, 즉시무효화, revocation, 면접답변]
topic: "쿠키는 저장·전송 수단이고, 세션과 토큰은 인증 상태를 서버에 두느냐(stateful) 클라이언트가 들고 다니느냐(stateless)의 차이다"
summary: "HTTP 는 무상태라 요청 사이를 기억하지 못하고, 매 요청마다 비밀번호를 검증하는 건 일부러 느린 해시(bcrypt)를 반복하는 일이라 감당할 수 없다. 그래서 로그인 때 한 번만 검증하고 결과를 세션 ID 나 토큰으로 바꿔 들고 다닌다. 쿠키는 그 값을 담아 자동 전송하는 수단이라 세션·토큰과 같은 층위가 아니며, 토큰도 쿠키에 담을 수 있다. 쿠키는 자동 전송이라 CSRF 에, localStorage 는 JS 가 읽을 수 있어 XSS 에 취약하다."
contributors: [dongju]
source_refs:
  - https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies
  - https://datatracker.ietf.org/doc/html/rfc6265
updated: 2026-10-22
---

# 세션 vs 토큰 — 그리고 쿠키는 왜 비교 대상이 아닌가

## 왜 필요한가 — 무상태 HTTP + 비싼 검증

- HTTP 는 **무상태(Stateless)** 다. 서버는 1번 요청과 2번 요청이 같은 사람에게서 왔는지 프로토콜 차원에서 모른다.
- 상태를 유지하는 장치가 없으면 매 요청마다 아이디·비밀번호를 보내야 한다.
  1. 민감정보가 매번 오가며 **노출 기회**가 늘어난다.
  2. 서버는 매번 **비밀번호 해시 검증**을 반복한다. bcrypt 는 **일부러 느리게**(1회 수십~수백 ms) 설계됐으므로 매 요청 반복은 감당할 수 없다 ([[secret-hashing-algorithm-choice]]).
- → 로그인 때 **딱 한 번** 비싼 검증을 하고, 그 결과를 **세션 ID 나 토큰**으로 바꿔 들고 다닌다.

## 층위 구분 — 쿠키는 세션·토큰과 비교할 대상이 아니다

| 개념 | 정체 | 층위 |
|---|---|---|
| **쿠키** | 브라우저의 **저장 + 자동 전송 수단** | 운반 수단 |
| **세션** | 인증 상태를 **서버**에 두는 방식 (Stateful) | 상태 관리 방식 |
| **토큰** | 인증 상태를 **토큰 안에 담아 클라이언트**가 들고 다니는 방식 (Stateless) | 상태 관리 방식 |

- 세션 방식도 브라우저가 **세션 ID** 를 보내야 서버가 세션을 찾는다 → 보통 쿠키에 담는다 (`JSESSIONID`).
- 토큰도 쿠키에 담을 수 있다 → "쿠키 vs 토큰"은 어색한 비교다. 진짜 축은 **인증 상태를 어디에 보관하느냐**.

## 운반 방식 — 쿠키(자동) vs localStorage(수동)

```
서버 → 브라우저:  Set-Cookie: SESSION=abc; HttpOnly; Secure; SameSite=Lax
브라우저 → 서버:  Cookie: SESSION=abc          ← 해당 도메인 요청마다 '자동'

localStorage:    JS 가 직접 꺼내서
                 Authorization: Bearer <token>  ← '수동'
```

| 저장소 | 특징 | 취약한 공격 |
|---|---|---|
| 쿠키 | 브라우저가 **자동** 전송 | **CSRF** — 내 권한으로 원치 않는 요청이 나감 ([[csrf-conditions-and-defenses]]) |
| localStorage | **JS 가 읽을 수 있음** | **XSS** — 심어진 스크립트가 토큰을 읽어 감 ([[token-storage-httponly-cookie]]) |

## 트레이드오프 — 상태를 어디에 두느냐의 대가

| | 세션 (Stateful) | 토큰 (Stateless) |
|---|---|---|
| 서버 저장소 | 세션 저장소 필요 | 서명 검증만 (저장소 불필요) |
| 수평 확장 | 서버 간 세션 공유 문제 → 스티키 세션 / Redis 세션 | 어느 서버든 검증 가능 |
| 즉시 무효화 | 세션 삭제하면 끝 | 만료 전까지 유효 → 짧은 만료 + 리프레시 토큰, 블랙리스트 ([[refresh-token-rotation]]) |
| 노출되는 정보 | 세션 ID 는 의미 없는 난수 | JWT payload 는 누구나 읽음 ([[jwt-structure-signature]]) |

> [!WARNING]
> **오답 코너**
> - **"서버가 기억 못 하는 건 요청된 정보에만 응답하기 때문"** — 용어로 말할 것: **무상태성(Stateless)**. 요청과 요청 사이를 연결해 기억하지 않는다.
> - **"매번 로그인하면 리소스를 잡아먹는다"** — 무엇이? **일부러 느린 비밀번호 해시**를 매 요청 반복하는 CPU 비용 + 민감정보 노출 기회.
> - **"쿠키 vs 세션 vs 토큰"** — 쿠키는 운반 수단, 세션·토큰은 상태 관리 방식. 층위가 다르다.
> - **"세션 ID 는 쿠키나 storage 에 저장한다"(동등하게 나열)** — 쿠키는 `Cookie` 헤더로 **자동** 전송(CSRF 위험), localStorage 는 JS 가 `Authorization` 에 **수동**으로 담음(XSS 위험). 전송 방식이 달라 받는 공격이 다르다.

## 면접 답변

> "HTTP 는 무상태라 로그인 상태를 따로 유지해야 합니다. 매 요청마다 비밀번호를 검증하면 일부러 느린 bcrypt 를 계속 돌려야 하니, 로그인 때 한 번만 검증하고 결과를 세션 ID 나 토큰으로 들고 다닙니다. 쿠키는 그 값을 브라우저에 저장하고 자동으로 보내는 수단이고, 세션과 토큰은 인증 상태를 서버에 두느냐 클라이언트가 들고 다니느냐의 차이입니다. 그래서 토큰도 쿠키에 담을 수 있습니다."

## 복습 체크

- [ ] 쿠키·세션·토큰이 왜 필요한지 "무상태"와 "느린 해시" 두 단어로 설명할 수 있는가?
- [ ] 쿠키가 세션·토큰과 같은 층위가 아닌 이유는?
- [ ] 세션과 토큰을 가르는 진짜 축은?
- [ ] 쿠키와 localStorage 는 각각 어떻게 전송되고, 각각 어떤 공격에 취약한가?
- [ ] 세션 방식의 수평 확장 문제와 토큰 방식의 즉시 무효화 문제는?

## 관련

[[jwt-structure-signature]] · [[token-storage-httponly-cookie]] · [[csrf-conditions-and-defenses]] · [[refresh-token-rotation]] · [[secret-hashing-algorithm-choice]] · [[filter-interceptor-aop]]
