---
date: 2026-10-19
type: raw
author: dongju
tags: [필터, filter, 서블릿필터, servlet-filter, jakarta.servlet, FilterChain, 인터셉터, interceptor, HandlerInterceptor, preHandle, handler, HandlerMethod, HandlerMapping, DispatcherServlet, AOP, 프록시, proxy, JoinPoint, PreAuthorize, 서블릿컨테이너, 톰캣, tomcat, 스프링MVC, 스프링컨테이너, 요청흐름, request-flow, 인증, authentication, 인가, authorization, 스프링시큐리티, Spring-Security, SecurityFilterChain, AuthorizationFilter, SecurityContext, SecurityContextHolder, addFilterBefore, UsernamePasswordAuthenticationFilter, AuthenticationEntryPoint, JwtAuthenticationFilter, OncePerRequestFilter, 필터이중등록, FilterRegistrationBean, 리프레시토큰, refresh-token-rotation, 조용한로그아웃, study-board, 면접답변]
topic: 필터·인터셉터·AOP 는 요청 흐름의 어디서·누가·무엇을 들고 가로채느냐가 다르고, 인증은 시큐리티의 인가 검사(AuthorizationFilter)보다 앞이어야 해서 필터로 둔다 (study-board JwtAuthenticationFilter)
summary: 필터는 서블릿 컨테이너가 DispatcherServlet 앞에서, 인터셉터는 스프링 MVC 가 DispatcherServlet 과 컨트롤러 사이에서(handler 를 앎), AOP 는 스프링 컨테이너가 프록시로 빈 메서드 호출을 가로챈다. 스프링 시큐리티의 인가 검사는 필터 체인 안의 AuthorizationFilter 가 하므로, JWT 로 SecurityContext 를 채우는 인증은 그보다 앞의 필터여야 한다. 인터셉터로 옮기면 유효한 쿠키가 있어도 익명으로 판단돼 로그인 페이지로 튕긴다. 또 @Component 필터는 톰캣 필터 목록에도 자동 등록돼 두 번 실행될 수 있는데, 리프레시 토큰 rotation 과 맞물리면 두 번째 refresh 가 실패해 쿠키를 지우는 조용한 로그아웃이 생기며, OncePerRequestFilter 가 이를 막는다.
source: session
distilled: false
---

## 배운 개념

### 1. 위치와 소속

```
브라우저 → 톰캣 → [ 필터 ] → DispatcherServlet → [ 인터셉터 ] → 컨트롤러 → [ AOP ] → 서비스
```

| | 필터 | 인터셉터 | AOP |
|---|---|---|---|
| 소속 (누가 호출) | **서블릿 컨테이너(톰캣)** — `jakarta.servlet` 패키지, 스프링 없이도 존재 | **스프링 MVC** | **스프링 컨테이너(코어)** — 빈을 만드는 쪽이 프록시 생성. 웹 없는 배치 앱에서도 `@Transactional` 동작 |
| 위치 | DispatcherServlet **앞** | DispatcherServlet ↔ 컨트롤러 | 빈 메서드 호출 |
| 받는 것 | `request`, `response`, `FilterChain` | + **`handler`** | `JoinPoint`(메서드 인자, 대상 객체) |
| 할 수 있는 것 | 요청 차단, request/response 감싸기·교체 | 어느 컨트롤러 메서드로 가는지(어노테이션 포함) 알고 처리 | 메서드 단위 공통 관심사(트랜잭션, `@PreAuthorize`) |

```java
doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)   // 필터
preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)            // 인터셉터
@Around("...") Object around(ProceedingJoinPoint joinPoint)                                    // AOP
```

- `handler` = 이 요청을 처리할 컨트롤러 메서드(`HandlerMethod`). DispatcherServlet 이 `HandlerMapping` 으로 **결정한 뒤**라 인터셉터는 받을 수 있고, 필터 시점엔 **아직 미결정**이라 못 받는다.
- AOP 는 쿠키(HTTP)를 직접 다루지 않고, **컨트롤러가 이미 실행된 뒤** 서비스 호출을 가로챈다 → 인증엔 부적합. 메서드 단위 **인가**(`@PreAuthorize`)용.
- `@Component` 필터는 스프링이 **만들어서** 톰캣 필터 목록에 **등록**해 줄 뿐, **호출**은 서블릿 필터 체인이 한다.

### 2. 인증을 필터로 둔 이유 (study-board)

```java
// JwtAuthenticationFilter extends OncePerRequestFilter
// 쿠키의 액세스 토큰(JWT)을 검증해 SecurityContext 에 인증을 채운다. 없거나 만료면 리프레시 토큰으로 재발급.

// SecurityConfig
.authorizeHttpRequests(auth -> auth
        .requestMatchers("/login", ...).permitAll()
        .anyRequest().authenticated())      // ← AuthorizationFilter 가 검사 (필터 체인 안)
.exceptionHandling(h -> h.authenticationEntryPoint(unauthenticatedEntryPoint))
.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
```

- **인증(Authentication)** = 누구인지 확인 → `JwtAuthenticationFilter` 가 `SecurityContext` 를 채움.
- **인가(Authorization)** = 이 URL 에 들어가도 되는지 → `AuthorizationFilter` 가 `anyRequest().authenticated()` 검사.
- 인가 검사가 **필터 체인 안**이므로 인증은 그보다 **앞 필터**에서 끝나 있어야 한다 → `addFilterBefore` 로 시큐리티 체인 안쪽에 둠.
- 인터셉터로 옮기면: 유효한 JWT 쿠키로 `/planner` 접속 → `AuthorizationFilter` 시점에 `SecurityContext` 가 비어 있음(익명) → 위반 → `UnauthenticatedEntryPoint` 가 화면 이동 요청이라 **로그인 페이지로 리다이렉트**. 인터셉터는 실행될 기회도 없음.

면접 답변:

> "세 가지는 가로채는 위치와 관리 주체가 다릅니다. 필터는 서블릿 컨테이너가 DispatcherServlet 앞에서, 인터셉터는 스프링 MVC 가 DispatcherServlet 과 컨트롤러 사이에서, AOP 는 스프링 컨테이너가 프록시로 빈 메서드 호출을 가로챕니다. 인증을 필터로 한 이유는 스프링 시큐리티의 인가 검사(`AuthorizationFilter`)가 필터 체인 안에서 일어나기 때문입니다. 인터셉터나 AOP 에서 JWT 를 읽으면 그 전에 `SecurityContext` 가 비어 있어 로그인한 사용자도 막히고, AOP 는 컨트롤러가 이미 실행된 뒤라 시점도 늦습니다. 그래서 `addFilterBefore` 로 시큐리티 체인 안쪽, 인가 검사보다 앞에 넣었습니다."

### 3. 숨은 함정: `@Component` 필터의 이중 등록과 OncePerRequestFilter

- Spring Boot 는 `Filter` 타입 빈을 **톰캣 필터 목록에도 자동 등록** → `addFilterBefore` 와 합쳐 **두 군데** 등록.
- 그냥 `jakarta.servlet.Filter` 였다면 요청당 **2회** 실행. 액세스 토큰이 만료된 요청이라면:

```
1회차: request 의 옛 리프레시 토큰 → DB 에서 찾음 → 삭제(1회용 rotation) → 새 토큰 쌍 → response 에 Set-Cookie(새 토큰)
2회차: 같은 request(바뀌지 않음)를 다시 읽음 → 만료 액세스 토큰 → refresh() 재호출
       → 옛 리프레시 토큰이 DB 에 없음 → 실패 → authCookies.clear(response) → Set-Cookie(삭제)
브라우저: 뒤쪽 Set-Cookie 가 이김 → 쿠키 전부 삭제 → 다음 요청부터 로그아웃
```

- 이번 화면은 1회차가 `SecurityContext` 를 채워 정상으로 보이지만, 액세스 토큰 만료 때마다 **조용한 로그아웃**.
- `OncePerRequestFilter` 가 요청 속성에 "이미 실행됨"을 표시해 같은 요청의 재호출에서 `doFilterInternal` 을 건너뛴다 → 지금 코드는 안전.
- 더 명확하게: `FilterRegistrationBean#setEnabled(false)` 로 서블릿 자동 등록을 끈다 (현재 프로젝트엔 없음).

## 오답·헷갈린 점

- **전**: 필터를 소속 분류에서 누락 (MVC / 스프링 밖 = 서블릿 컨테이너·브라우저) → **후**: 필터는 **서블릿 스펙**, 톰캣이 관리·호출.
- **전**: AOP 는 스프링 MVC 소속 → **후**: **스프링 컨테이너(코어)**. 웹 없어도 동작.
- **전**: 인터셉터가 `handler` 를 받는 이유 = "MVC 에서 지원하는 기능이라서" → **후**: DispatcherServlet 이 handler 를 **결정한 뒤**라서. 필터 시점엔 아직 미결정.
- **전**: "필터(시큐리티 콘피그)에서 막힌다" → **후**: `SecurityConfig` 는 체인을 **조립하는 설정 클래스**. 막는 건 **`AuthorizationFilter`**, 이유는 `SecurityContext` 가 비어 있어서.
- **전**: "스프링 시큐리티가 필터에서 로그인 여부를 **인증**한다" → **후**: 필터 체인에서 하는 건 **인가** 검사. **인증**은 `JwtAuthenticationFilter` 가 그 앞에서.
- **전**: 이중 실행 시 두 번째에서 `refresh()` 는 "호출하지 않는다" → **후**: **호출된다.** 1회차는 `response` 에 썼고 2회차는 바뀌지 않은 `request` 를 읽으므로. 결과는 rotation 실패 → 쿠키 삭제.
- **전**: 인증을 필터에 둔 이유로 "스프링에서 필터에 인증을 두기를 권장" → **후**: 권위에 기대는 이유는 면접에서 "왜 권장하죠?"로 되돌아온다. 진짜 이유(`AuthorizationFilter` 보다 앞)를 말할 것. (같은 날 첫 인터뷰 "스프링이 권장해서요 빼고"와 같은 패턴)
- 👍 개선: 마지막 종합 답변에서 처음으로 **질문 두 개에 모두** 답하고 세 가지의 **소속을 이름으로** 정리. 남은 과제는 **인증/인가 용어 구분**.

## Q&A

- **Q. 필터·인터셉터·AOP 의 순서는?** A. 필터 → DispatcherServlet → 인터셉터 → 컨트롤러 → AOP(서비스 프록시) → 서비스.
- **Q. 각각 누가 관리하나?** A. 필터 = 서블릿 컨테이너, 인터셉터 = 스프링 MVC, AOP = 스프링 컨테이너.
- **Q. 필터는 왜 handler 를 못 받나?** A. handler 는 DispatcherServlet 이 HandlerMapping 으로 나중에 결정하므로.
- **Q. AOP 로 인증하면?** A. 컨트롤러 실행 후라 늦고 HTTP 를 직접 다루지 않는다. 메서드 단위 인가용.
- **Q. 인터셉터로 JWT 인증을 옮기면?** A. AuthorizationFilter 가 먼저 빈 SecurityContext 를 보고 막아, 유효 쿠키가 있어도 로그인 페이지로 리다이렉트.
- **Q. JwtAuthenticationFilter 는 왜 OncePerRequestFilter 를 상속했나?** A. @Component 라 톰캣에도 자동 등록돼 이중 실행될 수 있는데, 리프레시 토큰 rotation 과 맞물리면 두 번째 refresh 실패로 쿠키가 지워지는 조용한 로그아웃이 생긴다. 이를 요청당 1회 실행으로 막는다.

## 다음 학습

- `DelegatingFilterProxy` 와 `FilterChainProxy`: 시큐리티 필터 체인이 톰캣 필터 하나로 어떻게 끼워지나, 서블릿 필터 순서와 시큐리티 내부 필터 순서
- `OncePerRequestFilter` 의 "이미 실행됨" 판단 방식과 ERROR/ASYNC 디스패치(SSE 비동기 요청)에서의 동작 (`shouldNotFilterAsyncDispatch`, `shouldNotFilterErrorDispatch`)
- study-board 에 `FilterRegistrationBean#setEnabled(false)` 적용 여부 검토
- 인터셉터가 적합한 실제 사례 (컨트롤러 어노테이션 기반 처리, 공통 모델 속성)
- 연결: [[transactional-proxy-self-invocation]], [[refresh-token-rotation]], [[token-storage-httponly-cookie]], [[token-refresh-concurrency]], [[2026-10-19-singleton-bean-stateful]]
