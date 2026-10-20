---
type: refined
slug: filter-interceptor-aop
tags: [필터, filter, 서블릿필터, servlet-filter, jakarta.servlet, FilterChain, 인터셉터, interceptor, HandlerInterceptor, preHandle, handler, HandlerMethod, HandlerMapping, DispatcherServlet, AOP, 프록시, proxy, JoinPoint, PreAuthorize, 서블릿컨테이너, 톰캣, tomcat, 스프링MVC, 스프링컨테이너, 요청흐름, request-flow, 인증, authentication, 인가, authorization, 스프링시큐리티, Spring-Security, SecurityFilterChain, AuthorizationFilter, SecurityContext, addFilterBefore, AuthenticationEntryPoint, JwtAuthenticationFilter, OncePerRequestFilter, 필터이중등록, FilterRegistrationBean, 조용한로그아웃, 면접답변]
topic: "필터·인터셉터·AOP 는 요청 흐름의 어디서·누가·무엇을 들고 가로채느냐가 다르고, JWT 인증은 시큐리티의 인가 검사(AuthorizationFilter)보다 앞이어야 하므로 필터로 둔다"
summary: "필터는 서블릿 컨테이너가 DispatcherServlet 앞에서, 인터셉터는 스프링 MVC 가 DispatcherServlet 과 컨트롤러 사이에서(handler 를 앎), AOP 는 스프링 컨테이너가 프록시로 빈 메서드 호출을 가로챈다. 스프링 시큐리티의 인가 검사는 필터 체인 안의 AuthorizationFilter 가 하므로 SecurityContext 를 채우는 인증은 그보다 앞 필터여야 하며, 인터셉터로 옮기면 유효한 토큰이 있어도 익명으로 판단돼 막힌다. 또 @Component 필터는 톰캣에도 자동 등록돼 두 번 실행될 수 있어 OncePerRequestFilter 나 FilterRegistrationBean 으로 막는다."
contributors: [dongju]
source_refs:
  - https://docs.spring.io/spring-security/reference/servlet/architecture.html
  - https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet/handlermapping-interceptor.html
  - https://docs.spring.io/spring-boot/reference/web/servlet.html
updated: 2026-10-20
---

# 필터 · 인터셉터 · AOP

## 개념 정의

```
브라우저 → 톰캣 → [ 필터 ] → DispatcherServlet → [ 인터셉터 ] → 컨트롤러 → [ AOP ] → 서비스
```

| | 필터 | 인터셉터 | AOP |
|---|---|---|---|
| 소속 (누가 호출) | **서블릿 컨테이너(톰캣)** — `jakarta.servlet` 스펙, 스프링 없이도 존재 | **스프링 MVC** | **스프링 컨테이너(코어)** — 빈을 만드는 쪽이 프록시 생성. 웹 없는 배치에서도 동작 |
| 위치 | DispatcherServlet **앞** | DispatcherServlet ↔ 컨트롤러 | 빈 메서드 호출 |
| 받는 것 | `request`, `response`, `FilterChain` | + **`handler`** | `JoinPoint`(메서드 인자, 대상 객체) |
| 잘하는 일 | 요청 차단, request/response 감싸기·교체 | 어느 컨트롤러 메서드로 가는지(어노테이션 포함) 알고 처리 | 메서드 단위 공통 관심사(트랜잭션, `@PreAuthorize`) |

```java
doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)   // 필터
preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)            // 인터셉터
@Around("...") Object around(ProceedingJoinPoint joinPoint)                                    // AOP
```

## 작동 원리

### 위치가 정보를 결정한다

- `handler` = 이 요청을 처리할 컨트롤러 메서드(`HandlerMethod`). DispatcherServlet 이 `HandlerMapping` 으로 **결정한 뒤**라 인터셉터는 받고, 필터 시점엔 **아직 미결정**이라 못 받는다.
- AOP 는 HTTP(쿠키)를 직접 다루지 않고 **컨트롤러가 이미 실행된 뒤** 서비스 호출을 가로챈다 ([[transactional-proxy-self-invocation]]). 그래서 "누가 요청했나"(인증)보다 "이 메서드를 호출해도 되나"(메서드 단위 인가)에 쓴다.
- `@Component` 필터는 스프링이 **만들어서** 톰캣 필터 목록에 **등록**해 줄 뿐, **호출**은 서블릿 필터 체인이 한다.

### 인증을 필터로 두는 이유

```java
.authorizeHttpRequests(auth -> auth
        .requestMatchers("/login", ...).permitAll()
        .anyRequest().authenticated())      // ← AuthorizationFilter 가 검사 (필터 체인 안)
.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
```

- **인증(Authentication)** = 누구인지 → JWT 필터가 `SecurityContext` 를 채움.
- **인가(Authorization)** = 들어가도 되는지 → `AuthorizationFilter` 가 검사.
- 인가 검사가 **필터 체인 안**이므로, 인증은 그보다 **앞 필터**에서 끝나 있어야 한다.
- 인터셉터로 옮기면: 유효한 JWT 쿠키로 보호된 경로 접속 → `AuthorizationFilter` 시점에 `SecurityContext` 가 비어 있음(익명) → `AuthenticationEntryPoint` 가 로그인 페이지로 리다이렉트. 인터셉터는 실행될 기회도 없다.

## 트레이드오프 / 한계 — `@Component` 필터의 이중 등록

Spring Boot 는 `Filter` 타입 빈을 **톰캣 필터 목록에도 자동 등록**한다. `addFilterBefore` 로 시큐리티 체인에도 넣었다면 **두 군데** 등록 → 일반 `Filter` 면 요청당 **2회** 실행.

액세스 토큰 만료 시 리프레시 토큰으로 재발급하는 필터라면:

```
1회차: request 의 리프레시 토큰 → 1회용 회전(사용 즉시 폐기) → 새 토큰 쌍 → response 에 Set-Cookie(새 토큰)
2회차: 바뀌지 않은 같은 request 를 다시 읽음 → 같은 리프레시 토큰으로 재발급 시도 → 이미 폐기 → 실패
       → 쿠키 삭제 Set-Cookie
브라우저: 뒤쪽 Set-Cookie 가 이김 → 쿠키 전부 삭제 → 다음 요청부터 로그아웃
```

이번 화면은 1회차가 `SecurityContext` 를 채워 정상으로 보이지만, 액세스 토큰이 만료될 때마다 **조용한 로그아웃**이 생긴다 ([[refresh-token-rotation]]).

- `OncePerRequestFilter` — 요청 속성에 "이미 실행됨"을 표시해 같은 요청의 재호출에서 `doFilterInternal` 을 건너뛴다.
- 더 명확하게 — `FilterRegistrationBean#setEnabled(false)` 로 서블릿 자동 등록 자체를 끈다.

> [!WARNING]
> **오답 코너**
> - **필터를 스프링 소속으로 착각** → 필터는 **서블릿 스펙**, 톰캣이 관리·호출.
> - **"AOP 는 스프링 MVC 소속"** → **스프링 컨테이너(코어)**. 웹 없어도 동작.
> - **"인터셉터가 handler 를 받는 건 MVC 가 지원해서"** → DispatcherServlet 이 handler 를 **결정한 뒤**라서. 필터 시점엔 미결정.
> - **"SecurityConfig(필터)가 막는다"** → `SecurityConfig` 는 체인을 **조립하는 설정**. 막는 건 **`AuthorizationFilter`**.
> - **"시큐리티가 필터에서 로그인 여부를 인증한다"** → 필터 체인에서 하는 건 **인가** 검사. **인증**은 그 앞의 JWT 필터.
> - **"이중 실행 시 두 번째에는 재발급을 호출하지 않는다"** → **호출된다.** 1회차는 `response` 에 썼고 2회차는 바뀌지 않은 `request` 를 읽는다.
> - **"스프링이 필터에 인증을 두라고 권장해서"** → 권위는 이유가 아니다. 진짜 이유는 **`AuthorizationFilter` 보다 앞**이어야 하기 때문.

## 복습 체크

- [ ] 필터·인터셉터·AOP 의 순서와 각각의 관리 주체를 말할 수 있다
- [ ] 인터셉터만 `handler` 를 받는 이유를 DispatcherServlet 의 역할로 설명할 수 있다
- [ ] AOP 가 인증에 맞지 않는 이유 두 가지를 말할 수 있다
- [ ] 인증과 인가를 구분하고, 각각 어느 필터가 하는지 말할 수 있다
- [ ] JWT 인증을 인터셉터로 옮기면 로그인한 사용자에게 무슨 일이 생기는지 흐름으로 설명할 수 있다
- [ ] `@Component` 필터가 두 번 실행되는 이유와, 리프레시 토큰 회전과 맞물려 생기는 버그, 두 가지 방지책을 말할 수 있다

## 관련

[[transactional-proxy-self-invocation]] · [[refresh-token-rotation]] · [[token-storage-httponly-cookie]] · [[exception-status-code-mapping]] · [[conditional-on-property]]
