---
type: refined
slug: sop-vs-cors
tags: [SOP, 동일출처정책, same-origin-policy, CORS, 교차출처리소스공유, cross-origin-resource-sharing, 출처, origin, 오리진, 사이트, site, same-site, eTLD+1, 등록가능도메인, registrable-domain, 서브도메인, subdomain, 읽기vs보내기, read-vs-send, preflight, 프리플라이트, OPTIONS, 단순요청, simple-request, Access-Control-Allow-Origin, allowedOrigins, localStorage, 출처별분리, CSRF, 면접답변]
topic: "SOP 는 다른 출처 데이터의 '읽기'만 막고 '보내기'는 막지 않으며, CORS 는 그 읽기 제한에 예외를 허락하는 장치다 — 그래서 CORS 는 CSRF 방어가 아니다"
summary: "SOP 는 브라우저가 다른 출처(프로토콜+호스트+포트)의 응답·localStorage 를 JS 가 읽지 못하게 하는 정책이지만 요청을 보내는 것은 막지 않는다. CORS 는 서버가 '이 출처는 우리 응답을 읽어도 된다'고 SOP 에 예외를 허락하는 장치다. 폼 제출 같은 단순 요청은 preflight 없이 서버에 도달하므로 CORS 설정으로 CSRF 를 막을 수 없다. 출처와 사이트(eTLD+1)는 다른 단위로, www.bank.com 과 blog.bank.com 은 다른 출처지만 같은 사이트다."
contributors: [dongju]
source_refs:
  - https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy
  - https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
  - https://developer.mozilla.org/en-US/docs/Glossary/Site
updated: 2026-10-22
---

# SOP 와 CORS — 읽기와 보내기

## 개념 정의

| | 막는 것 | 막지 못하는 것 |
|---|---|---|
| **SOP** (동일 출처 정책) | 다른 출처 데이터를 JS 가 **읽기** (응답 본문, localStorage, 다른 출처 DOM) | 요청 **보내기** |
| **CORS** | 아무것도 막지 않는다. SOP 에 **예외를 허락**하는 장치 ("이 출처는 우리 응답을 읽어도 된다") | — |

## 작동 원리

### 출처(origin) vs 사이트(site)

- **출처** = 프로토콜 + 호스트 + 포트. SOP 의 단위.
- **사이트** = 등록 가능한 도메인(**eTLD+1**). SameSite 쿠키의 단위.

| 비교 | 같은 출처? | 같은 사이트? |
|---|---|---|
| `https://www.bank.com` vs `https://blog.bank.com` | ❌ 호스트가 다름 | ⭕ 둘 다 `bank.com` |
| `https://bank.com` vs `https://evil.com` | ❌ | ❌ |

이 단위 차이가 SameSite 와 CSRF 토큰을 **두 겹**으로 쓰는 이유다 ([[csrf-conditions-and-defenses]]).

### 요청은 어떻게 서버에 닿는가

- **단순 요청**(폼 제출: GET/POST + `application/x-www-form-urlencoded` 등, 커스텀 헤더 없음)은 **preflight 없이 바로 서버에 도달**한다. 브라우저는 응답을 JS 에 넘길지(읽기)만 CORS 헤더로 판단한다.
- JSON 이나 커스텀 헤더(`Authorization`) 요청은 **preflight(OPTIONS)** 가 먼저 가고, 허락이 없으면 본 요청을 보내지 않는다.
- 공격자는 **단순 요청**을 쓰면 된다 → CORS 는 공격 경로에 아예 끼어들지 않는다.

### SOP 가 지켜 주는 것

- evil.com 의 JS 는 bank.com 의 **응답**(CSRF 토큰이 든 페이지)을 못 읽는다.
- evil.com 의 JS 는 bank.com 의 **localStorage**(액세스 토큰)를 못 읽는다.
- 두 가지 모두 "같은 컴퓨터에 저장돼서"가 아니라 **출처별 분리** 때문이다. 같은 브라우저 안에서 evil.com 스크립트도 돈다.

## 트레이드오프 / 한계

- **CORS `allowedOrigins` 를 좁혀도 CSRF 는 막히지 않는다** — 읽기 허락 범위를 정할 뿐, 단순 요청 도달은 그대로다.
- 반대로 요청의 `Origin` 을 그대로 반사해 credentials 와 함께 허락하면, 아무 출처나 **인증된 응답을 읽을 수 있게** 되어 정보 유출이 생긴다. (`*` 와 credentials 조합은 브라우저가 거부한다.) CORS 설정 실수는 CSRF 가 아니라 **읽기 유출** 문제다.
- SOP 는 **XSS 앞에서 무력**하다 — 주입된 스크립트는 같은 출처에서 실행된다 ([[token-storage-httponly-cookie]]).

> [!WARNING]
> **오답 코너**
> - **"CORS 는 요청을 보내는 것을 막는다, 폼에도 검사가 적용된다"** — SOP·CORS 는 **읽기**만 다룬다. 폼 제출은 CORS 검사 없이 서버에 도달한다. **CORS 는 CSRF 방어가 아니다.**
> - **"localStorage 를 다른 사이트가 못 읽는 건 내 컴퓨터에 저장되기 때문"** — 이유는 **출처별 분리(SOP)**.
> - **"서브도메인이나 www 가 붙으면 다른 사이트"** — 다른 **출처**일 뿐 같은 **사이트**(eTLD+1)다.

## 면접 답변

> "SOP 는 다른 출처의 응답이나 저장소를 자바스크립트가 읽지 못하게 막는 브라우저 정책이고, 요청을 보내는 것 자체는 막지 않습니다. CORS 는 서버가 특정 출처에게 그 읽기를 허락해 주는 예외 장치입니다. 그래서 폼 제출 같은 단순 요청은 CORS 와 상관없이 서버에 도달하고, CORS 설정으로는 CSRF 를 막을 수 없습니다."

## 복습 체크

- [ ] SOP 가 막는 것과 막지 않는 것을 한 문장으로 말할 수 있는가?
- [ ] CORS 가 "막는 장치"가 아니라 "허락하는 장치"라는 뜻은?
- [ ] 출처와 사이트의 정의는? `www.bank.com` 과 `blog.bank.com` 은 각각 같은가?
- [ ] preflight 가 가는 요청과 안 가는 요청의 차이, 그리고 공격자가 어느 쪽을 쓰는가?
- [ ] evil.com 이 CSRF 토큰과 localStorage 토큰을 못 읽는 공통 이유는?

## 관련

[[csrf-conditions-and-defenses]] · [[token-storage-httponly-cookie]] · [[session-vs-token-auth]]
