---
date: 2026-10-20
type: raw
author: dongju
tags: [CSRF, 사이트간요청위조, cross-site-request-forgery, CSRF토큰, csrf-token, synchronizer-token, XSRF-TOKEN, X-XSRF-TOKEN, CsrfFilter, Spring-Security, SameSite, Strict, Lax, 쿠키자동전송, cookie-auto-attach, Authorization헤더, Bearer, localStorage, JWT, 토큰방식, token-based-auth, SOP, 동일출처정책, same-origin-policy, CORS, 교차출처리소스공유, preflight, 단순요청, simple-request, 출처, origin, 사이트, site, eTLD+1, 서브도메인, subdomain, 심층방어, defense-in-depth, 상태변경요청, state-changing-request, study-board, 면접답변]
topic: "CSRF 는 세 조건(유효한 인증 정보·브라우저 자동 첨부·요청 값 예측 가능)이 겹칠 때 성립하고, SameSite 는 조건 2 를, CSRF 토큰은 조건 3 을 깬다. 토큰 방식이 안전한 이유는 토큰이 아니라 전송 방식(헤더 수동 첨부)이고, CORS 는 CSRF 방어가 아니다"
summary: "CSRF 는 다른 사이트에서 시작된 요청에 브라우저가 인증 쿠키를 자동으로 붙이는 점을 악용해 피해자 권한으로 상태를 바꾸는 공격이며, 공격자는 응답을 볼 필요가 없다. 성립 조건은 ① 유효한 인증 정보 ② 브라우저 자동 첨부 ③ 요청 값 예측 가능. SameSite 는 다른 사이트발 요청에 쿠키를 빼고 보내 ②를, CSRF 토큰은 공격자가 직접 넣어야 하지만 SOP 때문에 읽을 수 없는 값을 요구해 ③을 깬다. Authorization 헤더 토큰은 자동 첨부가 없어 안전하지만 쿠키에 담으면 다시 노출된다. SOP 는 읽기만 막고 보내기는 막지 않으며 CORS 는 그 예외를 허락하는 장치라 CSRF 를 못 막는다. SameSite 는 사이트(eTLD+1) 단위라 형제 서브도메인을 구별 못 하므로 출처 단위 SOP 에 기대는 CSRF 토큰과 두 겹으로 쓴다."
source: session
distilled: true
distilled_at: 2026-10-22
---

## 배운 개념

### 1. CSRF 공격 시나리오

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
- 공격자는 ⑤의 응답을 볼 필요가 없다. 목표는 응답 읽기가 아니라 **서버 상태 변경**(송금·비밀번호 변경·이메일 변경). GET 이면 `<img src="https://bank.com/transfer?to=...">` 한 줄로도 된다.

### 2. 성립 조건 세 가지와 방어

| 조건 | 깨는 장치 | 누가 무엇을 하나 |
|---|---|---|
| 1. 피해자가 유효한 인증 정보를 가짐 | — | 깨기 어려움 |
| 2. 브라우저가 인증 정보를 **자동** 첨부 | SameSite 쿠키 / Authorization 헤더 토큰 | **브라우저**가 다른 사이트발 요청에 쿠키를 **빼고** 보냄 → 익명 요청 → 401/403 |
| 3. 공격자가 요청의 모든 값을 **예측** 가능 | CSRF 토큰 | **서버**가 랜덤 값을 발급·기억하고, 요청에 담긴 값과 비교 |

- 조건 3 의 "예측 불가능한 값"은 **공격자가 요청에 직접 넣어야 하는 값**이어야 한다. 쿠키 속 JWT 는 예측 불가능해도 브라우저가 자동으로 붙여 주므로 방어가 안 된다.

### 3. SameSite 는 요청을 막지 않는다

```
SameSite 없음:   POST /transfer   Cookie: ACCESS_TOKEN=eyJ...   → 피해자로 인증 → 송금
SameSite=Strict: POST /transfer   (쿠키 없음)                    → 인증 실패 401/403
```

- 요청은 서버에 **도달**한다. 브라우저가 쿠키를 빼서 **익명 요청**이 될 뿐.

### 4. CSRF 토큰의 동작

```
① 서버가 랜덤 CSRF 토큰을 발급하고 자기 쪽(세션 등)에 기억
② 우리 페이지에 담아 내려보냄
   - 폼: <input type="hidden" name="_csrf" value="x7f...">
   - SPA: JS 가 읽을 수 있는 쿠키(XSRF-TOKEN)나 meta 태그 → JS 가 X-XSRF-TOKEN 헤더에 직접 담음
③ 상태 변경 요청(POST/PUT/DELETE)이 오면 요청의 토큰과 기억한 토큰을 비교
④ 없거나 다르면 403      ← Spring Security CsrfFilter 기본 동작
```

공격자가 실패하는 이유는 **두 겹**:
- **직접 담아야 한다**: 브라우저가 자동으로 붙여 주지 않는다.
- **값을 알 수 없다**: 토큰은 bank.com 페이지 안에 있고, **SOP** 때문에 evil.com 은 그 응답을 읽지 못한다.

### 5. 토큰 방식이 CSRF 를 막는 이유 — 토큰이 아니라 전송 방식

- JWT 를 localStorage 에 두고 JS 가 `Authorization: Bearer ...` 로 **직접** 붙이면, ④에서 브라우저가 자동 첨부하지 않아 공격 요청에 인증 정보가 없다.
- evil.com 의 JS 가 bank.com 의 localStorage 를 읽어 붙일 수도 없다 — localStorage 는 **출처별로 분리**(SOP).
- ⚠️ 토큰을 **쿠키**에 담으면(`Set-Cookie: ACCESS_TOKEN=...; HttpOnly`) 다시 자동 첨부 → CSRF 에 노출. 그래서 쿠키 방식이면 SameSite·CSRF 토큰이 필요하다.

### 6. SOP · CORS 의 범위

| | 막는 것 | 막지 못하는 것 |
|---|---|---|
| SOP | 다른 출처 데이터를 JS 가 **읽기** (응답, localStorage) | 요청 **보내기** |
| CORS | 막지 않음. SOP 에 **예외를 허락**하는 장치("이 출처는 우리 응답을 읽어도 된다") | — |

- 폼 제출 같은 **단순 요청**은 CORS 검사·preflight 없이 서버에 도달한다. JSON 이나 커스텀 헤더(`Authorization`) 요청은 preflight(OPTIONS)가 먼저 가고 허락이 없으면 본 요청을 안 보내지만, 공격자는 단순 요청을 쓴다.
- → **CORS `allowedOrigins` 설정은 CSRF 방어가 아니다.**

### 7. 출처(origin) vs 사이트(site), 그리고 두 겹 방어

| 비교 | 같은 출처? | 같은 사이트? |
|---|---|---|
| `https://www.bank.com` vs `https://blog.bank.com` | ❌ 호스트가 다름 | ⭕ 둘 다 `bank.com` (eTLD+1) |
| `https://bank.com` vs `https://evil.com` | ❌ | ❌ |

- 출처 = 프로토콜 + 호스트 + 포트. 사이트 = 등록 가능한 도메인(eTLD+1).
- 시나리오: 공격자가 `blog.bank.com` 게시판에 스크립트를 심어 `www.bank.com/transfer` 로 POST.
  - (a) `SameSite=Strict` 쿠키 → **실린다** (같은 사이트).
  - (b) CSRF 토큰 → **통과 못 한다** (다른 출처라 SOP 로 www 페이지의 토큰을 못 읽음).
- SameSite 는 사이트 단위라 같은 사이트 안의 다른 출처를 구별 못 하고, CSRF 토큰은 출처 단위 SOP 에 기대 그 틈을 메운다 → **두 겹**.
- ⚠️ 단서: CSRF 토큰 쿠키를 `Domain=bank.com` 처럼 상위 도메인으로 내리면 서브도메인이 읽거나 덮어써 방어가 약해진다. 발급 호스트로 한정할 것.

### 8. study-board

- 메인 `SecurityFilterChain` 은 CSRF 활성 — 토큰 없으면 403 ([[conditional-on-property]] 노트 기록).
- 리프레시 토큰 쿠키는 `HttpOnly; Secure; SameSite=Strict` ([[refresh-token-rotation]]).
- → 조건 2 는 SameSite, 조건 3 은 CSRF 토큰이 깬다. (CORS 설정이 아님)

### 면접 답변

> "CSRF 는 사용자가 로그인한 상태에서 악성 사이트가 우리 서버로 요청을 보내게 만들고, 브라우저가 그 요청에 인증 쿠키를 자동으로 붙여서 사용자 권한으로 송금 같은 상태 변경이 일어나는 공격입니다. 공격자는 응답을 볼 필요가 없습니다. 성립하려면 유효한 인증 정보, 브라우저의 자동 첨부, 예측 가능한 요청 값이라는 세 조건이 필요한데, SameSite 는 다른 사이트발 요청에서 쿠키를 빼서 두 번째 조건을 깨고, CSRF 토큰은 공격자가 직접 넣어야 하지만 SOP 때문에 읽을 수 없는 값을 요구해서 세 번째 조건을 깹니다. JWT 를 Authorization 헤더로 보내는 방식이 안전한 이유도 토큰 자체가 아니라 브라우저가 자동으로 붙이지 않기 때문이라, 토큰을 쿠키에 담으면 다시 CSRF 대책이 필요합니다. CORS 는 응답 읽기를 허락하는 장치라 CSRF 방어가 아니고, SameSite 는 서브도메인을 같은 사이트로 보기 때문에 CSRF 토큰과 함께 두 겹으로 둡니다."

## 오답·헷갈린 점

- **전**: "CSRF 는 크로스 사이트 요청을 **막아준다**" → **후**: CSRF 는 **공격**의 이름. 막는 쪽은 SameSite·CSRF 토큰.
- (인터뷰어 정정 👍) 시나리오에 "bank.com 탭을 닫지 않은 채"를 조건으로 넣었는데, 쿠키는 탭이 아니라 브라우저에 남으므로 불필요하다고 스스로 지적.
- **전**: 성립 조건을 단계별로만 나열, ③을 "폼이 필요" → **후**: 폼은 수단일 뿐(`<img>` 로도 됨). 조건은 **요청 값을 예측할 수 있다**.
- **전**: localStorage 를 다른 사이트가 못 읽는 이유 = "해당 컴퓨터 내에 저장되기 때문" → **후**: evil.com 의 JS 도 같은 컴퓨터·브라우저에서 돈다. 이유는 **출처별 분리(SOP)**.
- **전**: evil.com 이 CSRF 토큰을 못 얻는 이유 = "주소별로 발급되기 때문" → **후**: **SOP 가 다른 출처 응답 읽기를 막기 때문**. localStorage 질문과 같은 답.
- **전**: "CORS 는 요청을 보내는 것을 막는다, 폼에도 검사가 적용된다" → **후**: SOP·CORS 는 **읽기**만 다룬다. 폼 제출은 CORS 검사 없이 서버에 도달. **CORS 는 CSRF 방어가 아니다.** (가장 중요한 오개념)
- **전**: study-board 의 CSRF 방어 = "security config 에서 허용 주소를 명시" → **후**: 실제 방어는 SameSite=Strict + CSRF 토큰(CsrfFilter).
- **전**: "토큰을 동일 출처에서만 동작하도록, 토큰 관리는 서버에서" → **후**: 장치 이름으로. 어느 토큰(인증 JWT vs CSRF 토큰)인지 구분.
- **전**: 장치 ① JWT → 조건 3, 장치 ② CSRF 토큰 → 조건 2 ("동일 사이트인지 확인해 차단") → **후**: JWT 쿠키는 브라우저가 자동으로 붙이므로 조건 3 을 못 깬다. 같은 사이트인지 보고 쿠키를 뺄지 정하는 건 **브라우저(SameSite)**, 요청의 토큰을 비교하는 건 **서버(CSRF 토큰)** — 주체가 다르다. 최종: SameSite → 2, CSRF 토큰 → 3.
- **전**: SameSite=Strict 가 "다른 사이트의 요청을 **막는다**" → **후**: 요청은 도달하고, 쿠키만 빠져 **익명 요청**이 된다.
- **전**: CSRF 토큰으로 공격자가 실패하는 이유 = "자동으로 넘기지 않아 직접 담아야 하기 때문"(한 겹만) → **후**: 직접 담아야 하는데 **SOP 때문에 값을 알 수 없다**까지 짝으로.
- **전**: "서브도메인이나 www 를 붙이면 모두 다른 사이트" → **후**: SameSite 의 사이트는 eTLD+1 → 서브도메인은 **같은 사이트**. 그래서 SameSite 만으로 부족.
- ⚠️ 반복 테마: 보안 장치를 **"막는다"에서 멈춤**(CORS, SameSite). 직전 인터뷰("안전장치", "막힌다")와 같은 패턴. 답할 때 **누가 / 무엇을 / 어떤 조건을** 깨는지 세 가지를 붙일 것. 또 처음 질문이 추상적일 때 막혔다가 구체 시나리오가 주어지자 답이 정확해짐 → 스스로 공부할 때도 시나리오부터 그려 볼 것.

## Q&A

- **Q. CSRF 란?** A. 브라우저의 인증 쿠키 자동 첨부를 악용해, 다른 사이트에서 피해자 권한으로 상태 변경 요청을 보내게 하는 공격. 응답은 필요 없다.
- **Q. 성립 조건은?** A. 유효한 인증 정보, 브라우저 자동 첨부, 예측 가능한 요청 값.
- **Q. 토큰 방식은 왜 CSRF 를 막나?** A. Authorization 헤더에 JS 가 직접 붙여 자동 첨부가 없고, 다른 출처는 SOP 때문에 localStorage 를 못 읽는다. 쿠키에 담으면 다시 취약.
- **Q. CSRF 토큰은 어떻게 동작하나?** A. 서버가 랜덤 값을 발급·기억하고 페이지에 담아 보냄. 상태 변경 요청에 그 값이 있어야 통과. 공격자는 직접 넣어야 하는데 SOP 때문에 값을 모른다.
- **Q. CORS 로 CSRF 를 막을 수 있나?** A. 없다. CORS 는 응답 읽기를 허락하는 장치이고, 폼 같은 단순 요청은 서버에 그대로 도달한다.
- **Q. SameSite 만으로 충분한가?** A. 아니다. 사이트(eTLD+1) 단위라 형제 서브도메인이 뚫리면 쿠키가 실린다. 출처 단위인 CSRF 토큰과 두 겹으로.

## 다음 학습

- Double Submit Cookie 패턴과 Synchronizer Token 의 차이, 서브도메인 cookie tossing 공격
- Spring Security 에서 JWT 쿠키 방식일 때 `CookieCsrfTokenRepository.withHttpOnlyFalse()` + SPA 연동, CSRF 를 끄면 안 되는 조건
- study-board 실제 코드에서 액세스 토큰 쿠키의 SameSite 값(Lax/Strict)과 CSRF 토큰 전달 방식 확인
- preflight 가 발생하는 조건(단순 요청 판정: 메서드·Content-Type·헤더)과 `Origin`/`Referer` 헤더 검증
- 연결: [[token-storage-httponly-cookie]], [[refresh-token-rotation]], [[conditional-on-property]], [[2026-10-20-cookie-session-token-cookie-attributes]]
