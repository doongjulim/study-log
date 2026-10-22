---
type: refined
slug: csrf-conditions-and-defenses
tags: [CSRF, XSRF, 사이트간요청위조, 크로스사이트요청위조, cross-site-request-forgery, 성립조건, CSRF토큰, csrf-token, synchronizer-token, 동기화토큰, XSRF-TOKEN, X-XSRF-TOKEN, _csrf, CsrfFilter, CookieCsrfTokenRepository, Spring-Security, SameSite, Strict, Lax, 쿠키자동전송, cookie-auto-attach, ambient-authority, Authorization헤더, Bearer, localStorage, JWT쿠키, 토큰방식, token-based-auth, SOP, 동일출처정책, CORS, 출처, origin, 사이트, site, eTLD+1, 서브도메인, subdomain, 심층방어, defense-in-depth, 두겹방어, 상태변경요청, state-changing-request, 익명요청, study-board, 면접답변]
topic: "CSRF 는 세 조건(유효한 인증 정보·브라우저 자동 첨부·요청 값 예측 가능)이 겹칠 때 성립한다. SameSite 는 조건 2 를, CSRF 토큰은 조건 3 을 깨고, 둘은 사이트 vs 출처 단위라 두 겹으로 쓴다"
summary: "CSRF 는 다른 사이트에서 시작된 요청에 브라우저가 인증 쿠키를 자동으로 붙이는 점을 악용해 피해자 권한으로 상태를 바꾸는 공격이며, 공격자는 응답을 볼 필요가 없다. SameSite 는 브라우저가 다른 사이트발 요청에서 쿠키를 빼 익명 요청으로 만들어 조건 2 를, CSRF 토큰은 공격자가 직접 넣어야 하지만 SOP 때문에 읽을 수 없는 값을 서버가 요구해 조건 3 을 깬다. Authorization 헤더 토큰이 안전한 이유는 토큰이 아니라 수동 첨부라는 전송 방식이라, 토큰을 쿠키에 담으면 다시 취약해진다. SameSite 는 사이트(eTLD+1) 단위라 형제 서브도메인을 구별하지 못하므로 출처 단위 SOP 에 기대는 CSRF 토큰과 두 겹으로 둔다."
contributors: [dongju]
source_refs:
  - https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
  - https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html
  - https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#samesitesamesite-value
updated: 2026-10-22
---

# CSRF — 성립 조건과 방어

## 개념 정의 — 공격 시나리오

```
① 피해자가 bank.com 에 로그인 → Set-Cookie: SESSION=abc
② 피해자가 evil.com 에 접속 (bank.com 탭을 닫았어도 쿠키는 브라우저에 남아 있음)
③ evil.com 에 숨은 폼
   <form action="https://bank.com/transfer" method="POST">
     <input name="to" value="공격자계좌">
     <input name="amount" value="1000000">
   </form>
   <script>document.forms[0].submit()</script>
④ 브라우저가 bank.com 으로 POST 를 보내며 Cookie: SESSION=abc 를 자동 첨부
⑤ 서버는 유효한 쿠키를 보고 피해자의 정상 요청으로 판단해 송금
```

- **Cross-Site**: evil.com 에서 시작해 bank.com 으로. **Request Forgery**: 피해자가 보내지 않은 요청을 피해자 것처럼 위조.
- 공격자는 ⑤의 응답을 **볼 필요가 없다**. 목표는 응답 읽기가 아니라 **서버 상태 변경**(송금·비밀번호 변경·이메일 변경).
- 폼은 수단일 뿐이다. GET 으로 상태를 바꾸는 API 라면 `<img src="https://bank.com/transfer?to=...">` 한 줄로도 된다.

## 작동 원리 — 성립 조건 세 가지와 깨는 장치

| 조건 | 깨는 장치 | 누가 무엇을 하나 |
|---|---|---|
| 1. 피해자가 유효한 인증 정보를 가짐 | — | 깨기 어려움 |
| 2. 브라우저가 인증 정보를 **자동** 첨부 | **SameSite** 쿠키 / Authorization 헤더 토큰 | **브라우저**가 다른 사이트발 요청에서 쿠키를 **빼고** 보냄 |
| 3. 공격자가 요청의 모든 값을 **예측** 가능 | **CSRF 토큰** | **서버**가 랜덤 값을 발급·기억하고 요청 값과 비교 |

조건 3 의 "예측 불가능한 값"은 **공격자가 요청에 직접 넣어야 하는 값**이어야 한다. 쿠키 속 JWT 는 예측 불가능해도 브라우저가 자동으로 붙여 주므로 방어가 안 된다.

### SameSite 는 요청을 막지 않는다

```
SameSite 없음:   POST /transfer   Cookie: ACCESS_TOKEN=eyJ...   → 피해자로 인증 → 송금
SameSite=Strict: POST /transfer   (쿠키 없음)                    → 인증 실패 401/403
```

요청은 서버에 **도달**한다. 쿠키만 빠져 **익명 요청**이 될 뿐이다. (값별 동작은 [[token-storage-httponly-cookie]])

### CSRF 토큰의 동작

```
① 서버가 랜덤 CSRF 토큰을 발급하고 자기 쪽(세션 등)에 기억
② 우리 페이지에 담아 내려보냄
   - 폼: <input type="hidden" name="_csrf" value="x7f...">
   - SPA: JS 가 읽을 수 있는 쿠키(XSRF-TOKEN)나 meta 태그 → JS 가 X-XSRF-TOKEN 헤더에 직접 담음
③ 상태 변경 요청(POST/PUT/DELETE)이 오면 요청의 토큰과 기억한 토큰을 비교
④ 없거나 다르면 403      ← Spring Security CsrfFilter 기본 동작
```

공격자가 실패하는 이유는 **두 겹**이다.
- **직접 담아야 한다** — 브라우저가 자동으로 붙여 주지 않는다.
- **값을 알 수 없다** — 토큰은 bank.com 페이지 안에 있고, **SOP** 때문에 evil.com 은 그 응답을 읽지 못한다 ([[sop-vs-cors]]).

### 토큰 방식이 CSRF 를 막는 이유 — 토큰이 아니라 전송 방식

- JWT 를 localStorage 에 두고 JS 가 `Authorization: Bearer ...` 로 **직접** 붙이면, 브라우저가 자동 첨부하지 않아 공격 요청에 인증 정보가 없다.
- evil.com 의 JS 가 bank.com 의 localStorage 를 읽어 붙일 수도 없다 — localStorage 는 **출처별로 분리**(SOP).
- ⚠️ 토큰을 **쿠키**에 담으면(`Set-Cookie: ACCESS_TOKEN=...; HttpOnly`) 다시 자동 첨부 → CSRF 에 노출. 쿠키 방식이면 SameSite·CSRF 토큰이 필요하다.

## 트레이드오프 / 한계 — 왜 두 겹인가

SameSite 는 **사이트(eTLD+1)** 단위, CSRF 토큰은 **출처** 단위 SOP 에 기댄다 (정의는 [[sop-vs-cors]]).

시나리오: 공격자가 `blog.bank.com` 게시판에 스크립트를 심어 `www.bank.com/transfer` 로 POST.

| 장치 | 결과 | 이유 |
|---|---|---|
| `SameSite=Strict` 쿠키 | **실린다** | `blog.bank.com` 과 `www.bank.com` 은 같은 사이트(`bank.com`) |
| CSRF 토큰 | **통과 못 한다** | 다른 출처라 SOP 로 www 페이지의 토큰을 못 읽음 |

→ SameSite 의 틈(같은 사이트 안 다른 출처)을 CSRF 토큰이 메운다. **심층 방어(defense in depth)**.

- 단서: CSRF 토큰 쿠키를 `Domain=bank.com` 처럼 상위 도메인으로 내리면 서브도메인이 읽거나 덮어써(cookie tossing) 방어가 약해진다. 발급 호스트로 한정할 것.
- **CORS `allowedOrigins` 설정은 CSRF 방어가 아니다** — 폼 같은 단순 요청은 CORS 검사 없이 서버에 도달한다 ([[sop-vs-cors]]).

### study-board 적용

- 메인 `SecurityFilterChain` 은 CSRF 활성 — 토큰 없으면 403 (H2 콘솔 전용 체인만 해제: [[conditional-on-property]]).
- 리프레시 토큰 쿠키는 `HttpOnly; Secure; SameSite=Strict` ([[refresh-token-rotation]]).
- → 조건 2 는 SameSite, 조건 3 은 CSRF 토큰이 깬다. CORS 설정이 아니다.

> [!WARNING]
> **오답 코너**
> - **"CSRF 는 크로스 사이트 요청을 막아준다"** — CSRF 는 **공격**의 이름이다. 막는 쪽은 SameSite·CSRF 토큰.
> - **"피해자가 bank.com 탭을 열어 둔 상태여야 한다"** — 쿠키는 탭이 아니라 **브라우저**에 남는다.
> - **"조건 3 = 폼이 필요하다"** — 폼은 수단(`<img>` 로도 됨). 조건은 **요청 값을 예측할 수 있다**.
> - **"localStorage 는 그 컴퓨터에 저장되니 다른 사이트가 못 읽는다"** — evil.com 의 JS 도 같은 브라우저에서 돈다. 이유는 **출처별 분리(SOP)**.
> - **"evil.com 이 CSRF 토큰을 못 얻는 건 주소별로 발급되기 때문"** — **SOP 가 다른 출처 응답 읽기를 막기 때문**. localStorage 질문과 같은 답이다.
> - **"CORS 로 허용 주소를 명시하면 CSRF 가 막힌다"** — SOP·CORS 는 **읽기**만 다룬다. 폼 제출은 그대로 도달한다. (가장 중요한 오개념)
> - **"JWT 가 조건 3 을, CSRF 토큰이 조건 2 를 깬다"** — 쿠키 속 JWT 는 자동 첨부라 조건 3 을 못 깬다. 같은 사이트인지 보고 쿠키를 뺄지 정하는 건 **브라우저(SameSite) → 조건 2**, 요청 토큰을 비교하는 건 **서버(CSRF 토큰) → 조건 3**. 주체가 다르다.
> - **"SameSite=Strict 는 다른 사이트의 요청을 막는다"** — 요청은 도달하고 **쿠키만 빠진다**.
> - **"CSRF 토큰이 막는 이유 = 직접 담아야 해서"(한 겹만)** — 직접 담아야 하는데 **SOP 때문에 값을 모른다**까지 짝으로.
> - **"서브도메인은 모두 다른 사이트"** — 사이트는 eTLD+1 → 서브도메인은 **같은 사이트**. 그래서 SameSite 만으로 부족하다.
> - ⚠️ 답변 습관: 보안 장치를 **"막는다"에서 멈추지 말 것.** **누가 / 무엇을 / 어떤 조건을** 깨는지 세 가지를 붙인다.

## 면접 답변

> "CSRF 는 사용자가 로그인한 상태에서 악성 사이트가 우리 서버로 요청을 보내게 만들고, 브라우저가 그 요청에 인증 쿠키를 자동으로 붙여서 사용자 권한으로 송금 같은 상태 변경이 일어나는 공격입니다. 공격자는 응답을 볼 필요가 없습니다. 성립하려면 유효한 인증 정보, 브라우저의 자동 첨부, 예측 가능한 요청 값이라는 세 조건이 필요한데, SameSite 는 다른 사이트발 요청에서 쿠키를 빼서 두 번째 조건을 깨고, CSRF 토큰은 공격자가 직접 넣어야 하지만 SOP 때문에 읽을 수 없는 값을 요구해서 세 번째 조건을 깹니다. JWT 를 Authorization 헤더로 보내는 방식이 안전한 이유도 토큰 자체가 아니라 브라우저가 자동으로 붙이지 않기 때문이라, 토큰을 쿠키에 담으면 다시 CSRF 대책이 필요합니다. CORS 는 응답 읽기를 허락하는 장치라 CSRF 방어가 아니고, SameSite 는 서브도메인을 같은 사이트로 보기 때문에 CSRF 토큰과 함께 두 겹으로 둡니다."

## 복습 체크

- [ ] CSRF 성립 조건 세 가지를 말하고, 각 방어 장치가 어느 조건을 깨는지 짝지을 수 있는가?
- [ ] 공격자가 응답을 볼 필요가 없는 이유는?
- [ ] SameSite=Strict 쿠키일 때 공격 요청은 서버에 도달하는가? 도달하면 어떻게 되는가?
- [ ] CSRF 토큰으로 공격자가 실패하는 두 겹의 이유는?
- [ ] Authorization 헤더 토큰이 CSRF 에 안전한 진짜 이유와, 그 안전이 깨지는 경우는?
- [ ] `blog.bank.com` 에서 `www.bank.com` 으로의 공격을 SameSite 와 CSRF 토큰이 각각 어떻게 처리하는가?
- [ ] CORS 설정이 CSRF 방어가 아닌 이유는?

## 관련

[[sop-vs-cors]] · [[token-storage-httponly-cookie]] · [[session-vs-token-auth]] · [[refresh-token-rotation]] · [[conditional-on-property]] · [[filter-interceptor-aop]]
