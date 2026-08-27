---
type: refined
slug: handler-method-argument-resolver
tags: [ArgumentResolver, HandlerMethodArgumentResolver, PageableHandlerMethodArgumentResolver, PageableDefault, PageRequest, 파라미터바인딩, RequestParam, Model, Spring-MVC, DispatcherServlet, 핸들러어댑터, 자동주입, 커스텀ArgumentResolver, 로그인유저주입]
topic: 컨트롤러 메서드 파라미터가 저절로 채워지는 메커니즘과 Pageable 이 만들어지는 지점
summary: Spring MVC 는 컨트롤러 호출 직전에 파라미터마다 담당 ArgumentResolver 를 찾아 값을 채운다. Pageable 도 전용 리졸버가 쿼리스트링을 읽어 PageRequest 로 만들어 주는 것이며, 직접 만들어 확장할 수 있는 지점이다.
contributors: [dongju]
source_refs:
  - https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-methods/arguments.html
updated: 2026-08-25
---

# HandlerMethodArgumentResolver

## 개념 정의

```java
@GetMapping("/posts")
public String list(@RequestParam(required = false) String kw,
                   @PageableDefault(size = 10) Pageable pageable,
                   Model model) { ... }
```

컨트롤러에서 `Pageable` 을 직접 만든 적이 없는데 값이 채워져 있다.
Spring MVC 가 **컨트롤러 메서드를 호출하기 직전에, 파라미터마다 담당 리졸버를 찾아 값을 채워 넣기** 때문이다.

`@RequestParam`, `@PathVariable`, `Model`, `HttpServletRequest`, `@AuthenticationPrincipal` — 이 모두가
같은 계층의 같은 메커니즘이다. `Pageable` 만 특별한 것이 아니다.

## 작동 원리

```
DispatcherServlet
  → HandlerAdapter 가 호출할 컨트롤러 메서드 결정
  → 파라미터 목록을 순회
      → 각 파라미터마다 supportsParameter(param) == true 인 리졸버를 찾음
      → resolveArgument(...) 로 값 생성
  → 완성된 인자들로 컨트롤러 메서드 invoke
```

### `Pageable` 의 경우

Spring Boot 가 자동 구성으로 **`PageableHandlerMethodArgumentResolver`** 를 등록해 둔다.

1. 쿼리스트링의 `page` / `size` / `sort` 를 읽는다
2. `PageRequest`(= `Pageable` 구현체)를 만들어 파라미터에 넣는다
3. 값이 없으면 `@PageableDefault` 의 값으로 채운다

| 요청 | 결과 |
|---|---|
| `/posts?page=2&size=10` | `PageRequest.of(2, 10)` — **page 는 0부터** |
| `/posts?size=30` | `PageRequest.of(0, 30)` — 첫 페이지 |
| `/posts` | `@PageableDefault(size = 10)` → `PageRequest.of(0, 10)` |
| `/posts?sort=createdAt,desc` | 정렬 조건 포함 |

## 트레이드오프 / 한계

**확장 지점**: 직접 구현해 등록하면 반복되는 파라미터 조립을 컨트롤러 밖으로 빼낼 수 있다.

```java
public class LoginUserArgumentResolver implements HandlerMethodArgumentResolver {
    public boolean supportsParameter(MethodParameter p) {
        return p.hasParameterAnnotation(LoginUser.class);
    }
    public Object resolveArgument(...) { /* 세션/토큰에서 사용자 조회 */ }
}
```

`WebMvcConfigurer.addArgumentResolvers()` 로 등록한다.

**주의점**

- **`page` 는 0부터**다. 화면의 "1페이지"와 어긋나므로 뷰에서 `+1` 보정이 필요하다.
- **`size` 를 사용자가 지정할 수 있다.** `?size=1000000` 같은 요청이 그대로 통하면 대량 조회 공격이 된다.
  Spring Boot 는 `spring.data.web.pageable.max-page-size`(기본 2000) 로 상한을 두지만,
  서비스 성격에 맞게 낮추는 편이 안전하다.
- 리졸버가 무거운 작업(예: 매 요청 DB 조회)을 하면 **모든 요청에 비용이 붙는다.**
- 커스텀 리졸버가 예외를 던지면 컨트롤러 진입 전이라 디버깅 지점이 헷갈릴 수 있다.

> [!WARNING]
> **오답 코너**
> - **"`Pageable` 은 스프링이 알아서 채워주는 마법"** — `@RequestParam` 과 **동일 계층의 리졸버**가 채운다.
>   특별한 존재가 아니다.
> - **"`page=1` 이 첫 페이지"** — **0-based**. `page=1` 은 두 번째 페이지다.
> - **"`size` 는 서버가 정한 값으로 고정된다"** — 쿼리스트링으로 덮어쓸 수 있다. 상한 설정이 필요하다.
> - **"필터/인터셉터가 파라미터를 채운다"** — 그 둘은 요청 전후를 가로챌 뿐, 파라미터 바인딩은 리졸버 소관이다.

## 복습 체크

- [ ] 컨트롤러 파라미터가 채워지는 순서를 DispatcherServlet 부터 설명할 수 있는가?
- [ ] `Pageable` 을 채우는 구체적인 클래스 이름을 댈 수 있는가?
- [ ] `/posts?size=30` 이 몇 페이지로 해석되는가? 왜인가?
- [ ] 커스텀 리졸버를 만들 때 구현해야 하는 두 메서드는?
- [ ] `size` 를 사용자가 지정할 수 있다는 점의 위험과 대응은?

## 관련

[[db-pagination]] · [[spring-data-repository-proxy]] · [[derived-query-method]]
