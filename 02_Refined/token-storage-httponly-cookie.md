---
type: refined
slug: token-storage-httponly-cookie
topic: 토큰 저장 위치 — localStorage 와 HttpOnly 쿠키는 "어떤 공격을 받아들일지"의 선택이다
summary: localStorage 는 JS 가 읽을 수 있으므로 XSS 한 번에 토큰이 통째로 유출된다. HttpOnly 쿠키는 JS 가 값을 읽을 수 없어 유출을 막지만, 브라우저가 자동으로 실어 보내는 성질 때문에 CSRF 표면이 열린다. 다만 CSRF 는 SameSite 속성으로 닫을 수 있고 XSS 로 인한 값 유출은 localStorage 에서 닫을 방법이 없다는 비대칭이 선택을 결정한다.
tags: [토큰저장, token-storage, 저장위치, localStorage, 로컬스토리지, sessionStorage, 쿠키, cookie, HttpOnly, Secure, SameSite, Strict, Lax, Path, XSS, 크로스사이트스크립팅, cross-site-scripting, CSRF, 크로스사이트요청위조, cross-site-request-forgery, 리프레시토큰, refresh-token, 액세스토큰, access-token, 자동전송, ambient-authority]
contributors: [dongju]
source_refs:
  - https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
  - https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie
  - https://datatracker.ietf.org/doc/html/rfc9700
---

# 토큰 저장 위치 — localStorage vs HttpOnly 쿠키

## 핵심 명제

> **저장 위치를 고르는 것은 "어떤 공격을 받아들일지"를 고르는 것이다. 그리고 그 둘은 대칭이 아니다.**

- `localStorage` → **XSS 로 인한 값 유출**을 받아들인다. 이건 **닫을 방법이 없다.**
- `HttpOnly` 쿠키 → **CSRF 표면**을 받아들인다. 이건 **속성 하나로 닫힌다.**

닫을 수 있는 위험과 닫을 수 없는 위험 중 하나를 고르는 문제이므로, 답은 정해져 있다.

## 두 저장소의 성질

| | `localStorage` | `HttpOnly` 쿠키 |
|---|---|---|
| JS 가 값을 읽을 수 있는가 | **읽을 수 있다** | **읽을 수 없다** (`document.cookie`에도 안 나온다) |
| 요청에 실리는 방식 | 코드가 명시적으로 헤더에 넣는다 | **브라우저가 자동으로** 붙인다 |
| XSS 시 | 스크립트가 값을 통째로 읽어 외부로 전송 | 값 자체를 꺼낼 수 없음 |
| CSRF 시 | 자동 전송이 없으므로 애초에 성립 안 함 | **성립함** — 속성으로 막아야 함 |
| 전송 범위 제어 | 코드로 직접 제어 | `Path` / `Domain` 으로 선언적 제어 |

## 권장 형태

```
Set-Cookie: refreshToken=...; HttpOnly; Secure; SameSite=Strict; Path=/auth/refresh
```

| 속성 | 막는 것 |
|---|---|
| `HttpOnly` | JS 의 값 읽기 → **XSS 로 인한 토큰 유출** |
| `Secure` | 평문 HTTP 전송 → 네트워크 도청 |
| `SameSite=Strict` (또는 `Lax`) | 다른 사이트에서 시작된 요청에 쿠키 첨부 → **CSRF** |
| `Path=/auth/refresh` | 갱신 경로 외 요청에 쿠키 첨부 → 노출 빈도 자체를 축소 |

`Path` 를 걸면 `logo.png` 나 `/api/posts` 요청에는 리프레시 토큰이 **애초에 실려 가지 않는다.** 값이 오가는 횟수를 줄이는 것 자체가 방어다.

## CSRF 가 성립하는 원리

쿠키의 장점(브라우저가 알아서 실어 보냄)이 그대로 약점이 된다.

```html
<!-- 공격자 사이트 evil.com 에 심어둔 폼 -->
<form action="https://my-app.com/auth/refresh" method="post">…</form>
<script>document.forms[0].submit();</script>
```

브라우저는 **"my-app.com 으로 가는 요청"이라는 이유만으로** 그 도메인의 쿠키를 붙여 보낸다. 요청을 누가 시작했는지는 따지지 않는다. 사용자가 클릭한 적도, 그 사이트를 열어둔 것 외에 한 일도 없다.

이런 성질을 **ambient authority**(주변 권한)라 부른다 — 요청의 의도와 무관하게 자격 증명이 자동으로 따라붙는 구조.

`SameSite` 는 브라우저에게 **"다른 사이트에서 시작된 요청에는 이 쿠키를 붙이지 마"** 라고 선언하는 것이다.

| 값 | 동작 |
|---|---|
| `Strict` | 크로스사이트 요청에는 절대 첨부하지 않음. 외부 링크로 들어온 첫 진입에도 미첨부 |
| `Lax` | 톱레벨 GET 내비게이션에만 첨부. POST·서브리소스에는 미첨부 (요즘 브라우저 기본값) |
| `None` | 항상 첨부 — `Secure` 필수. 크로스도메인 구성에서만, CSRF 토큰 등 별도 방어와 함께 |

## HttpOnly 가 XSS 를 "해결"하지는 않는다

정확히 해둘 것: `HttpOnly` 가 막는 것은 **토큰 값의 유출**이지 XSS 자체가 아니다.

XSS 가 성립한 페이지에서 공격자 스크립트는 여전히 그 페이지 안에서 `fetch('/api/transfer', …)` 를 호출할 수 있고, 쿠키는 자동으로 붙는다. 즉 **값은 못 훔쳐도 그 세션으로 행동은 할 수 있다.**

차이는 **지속성과 범위**다.

| | 유출된 토큰 (localStorage) | HttpOnly 하에서의 XSS |
|---|---|---|
| 공격 지속 시간 | 토큰 만료/회전까지 — 사용자가 브라우저를 닫아도 계속 | 사용자가 그 페이지를 열어둔 동안만 |
| 공격 위치 | 공격자 서버에서 자유롭게 | 피해자 브라우저 안에서만 |
| 탐지 가능성 | 낮음 | 상대적으로 높음 (요청 패턴이 남음) |

> `HttpOnly` 는 XSS 를 막는 도구가 아니라 **XSS 의 피해 반경을 제한하는 도구**다. XSS 자체는 출력 이스케이프·CSP 로 막는다.

## 액세스 토큰은 어디에 두는가

리프레시 토큰과 액세스 토큰의 요구가 다르다.

| | 수명 | 저장 위치 | 근거 |
|---|---|---|---|
| 리프레시 토큰 | 길다 (일~주) | **`HttpOnly` 쿠키** | 유출 시 피해가 크고 길다. JS가 만질 이유가 없다 |
| 액세스 토큰 | 짧다 (분) | **JS 메모리 변수** | 요청 헤더에 넣어야 하므로 JS가 접근해야 한다. 새로고침 시 사라져도 갱신으로 복구된다 |

액세스 토큰을 `localStorage` 에 두는 것도 흔하지만, 메모리 변수에 두면 **탭을 닫는 순간 사라지고 XSS 유출 창도 그만큼 짧아진다.** 지속성은 리프레시 토큰(쿠키)이 담당하므로 사용자 경험은 그대로다.

> [!WARNING]
> **오답 코너**
>
> - **"쿠키는 다른 사이트에서 온 요청에는 안 실린다"** — **기본적으로 실린다.** 그래서 CSRF 가 성립한다. 안 실리게 만드는 것이 `SameSite` 속성이다.
> - **"HttpOnly 를 걸면 XSS 가 막힌다"** — 막히는 건 **값의 유출**이다. 공격자 스크립트는 그 페이지 안에서 여전히 인증된 요청을 보낼 수 있다. XSS 는 이스케이프·CSP 로 막는다.
> - **"localStorage 는 같은 오리진만 접근하니 안전하다"** — XSS 로 주입된 스크립트도 **같은 오리진에서 실행된다.** 오리진 격리는 XSS 앞에서 아무 보호도 되지 않는다.
> - **"쿠키는 CSRF 때문에 위험하니 localStorage 가 낫다"** — 비대칭을 놓친 판단이다. CSRF 는 `SameSite` 한 줄로 닫히지만, localStorage 의 XSS 유출은 닫을 방법이 없다.
> - **"토큰은 하나니까 저장 위치도 하나"** — 액세스 토큰과 리프레시 토큰은 수명과 접근 주체가 달라 저장 위치도 갈라야 한다.
> - **"`SameSite=None` 이 제일 호환성이 좋으니 그걸로"** — CSRF 방어를 스스로 끄는 선택이다. 크로스도메인이 불가피할 때만, 별도 CSRF 방어와 함께 쓴다.

## 복습 체크

- [ ] `localStorage` 와 `HttpOnly` 쿠키가 각각 받아들이는 위험이 무엇이고, 왜 그 둘이 대칭이 아닌가?
- [ ] `HttpOnly`, `Secure`, `SameSite`, `Path` 가 각각 무엇을 막는지 말할 수 있는가?
- [ ] CSRF 가 성립하는 원리를 공격자 폼 예시로 설명할 수 있는가?
- [ ] `SameSite=Strict` 와 `Lax` 의 차이는?
- [ ] `HttpOnly` 하에서 XSS 가 발생하면 공격자는 무엇을 할 수 있고 무엇을 할 수 없는가?
- [ ] 액세스 토큰과 리프레시 토큰의 저장 위치를 다르게 가져가는 근거는?

## 관련

- [[refresh-token-rotation]]
- [[token-refresh-concurrency]]
