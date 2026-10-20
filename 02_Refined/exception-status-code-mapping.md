---
type: refined
slug: exception-status-code-mapping
tags: [ControllerAdvice, RestControllerAdvice, ExceptionHandler, 예외처리, exception-handling, 전역예외처리, global-exception-handler, HTTP상태코드, http-status-code, 상태코드매핑, status-mapping, 4xx, 5xx, 400, 403, 404, 409, 422, 500, 클라이언트오류, 서버오류, 재시도, retry, 모니터링, monitoring, IllegalArgumentException, IllegalStateException, NumberFormatException, 표준예외, 커스텀예외, custom-exception, BusinessException, 예외계층, exception-hierarchy, 예외네이밍, 존재숨김, 정보노출, information-disclosure, log.error, HandlerExceptionResolver, DispatcherServlet, ExceptionTranslationFilter, AccessDeniedException, 면접답변]
topic: "@ControllerAdvice 의 상태 코드는 예외 \"타입\"이 아니라 실패의 \"의미\"(누구의 잘못·재시도)로 정한다 — 의도한 실패만 4xx, 나머지는 전부 5xx"
summary: "4xx 는 요청의 잘못이라 고치지 않으면 재시도해도 실패하고, 5xx 는 서버의 잘못이라 나중에 성공할 수 있으며 재시도·모니터링 알람의 대상이다. IllegalArgumentException·IllegalStateException 같은 자바 표준 예외는 의미가 아니라 현상을 담고 JDK·라이브러리도 던지므로, 그 타입으로 상태 코드를 정하면 규칙 위반이 404 가 되고 서버 버그가 4xx 로 위장된다. 의미가 드러나는 커스텀 BusinessException 계층만 4xx 로 보내고, 나머지 Exception 은 log.error + 500 + 일반 안내 문구로 처리한다. @ExceptionHandler 는 DispatcherServlet 이 호출하므로 시큐리티 필터의 예외는 닿지 않는다."
contributors: [dongju]
source_refs:
  - https://www.rfc-editor.org/rfc/rfc9110#name-status-codes
  - https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-advice.html
  - https://docs.spring.io/spring-security/reference/servlet/architecture.html#servlet-exceptiontranslationfilter
updated: 2026-10-20
---

# 예외 → HTTP 상태 코드 매핑

## 개념 정의

### @ControllerAdvice 가 해결한 것

없으면 컨트롤러 메서드마다 try-catch 로 같은 처리를 **반복**한다 → 오류 페이지 이름 하나 바꾸려면 전부 수정, try-catch 가 비즈니스 흐름을 가린다. 예외 처리라는 **공통 관심사를 한 곳으로** 모은 것이 `@ControllerAdvice`. HTML 오류 페이지용과 JSON 용을 나눌 땐 `@RestControllerAdvice(annotations = RestController.class)` + `@Order(HIGHEST_PRECEDENCE)` 로 REST 쪽이 먼저 잡게 한다.

### 상태 코드 기준: 누구의 잘못 + 재시도

| | 누구의 잘못 | 같은 요청 재시도 | 예 |
|---|---|---|---|
| **4xx** | 요청(클라이언트) | 고치지 않으면 **계속 실패** | 400 규칙 위반, 403 권한 없음, 404 없음, 409 충돌 |
| **5xx** | 서버 | 나중에 **성공할 수 있음** | 500 버그·인프라 장애 |

- 클라이언트·로드밸런서는 보통 5xx 만 재시도하고 모니터링 알람도 5xx 에 건다 → 서버 버그가 4xx 로 나가면 **장애가 지표에서 사라진다.**
- **형식 오류 ≠ 규칙 위반.** "종료 시간 < 시작 시간"은 형식은 멀쩡하고 규칙을 어긴 것. 둘 다 4xx(400, 또는 422 Unprocessable Content).
- **존재 숨김**: 권한 없는 비공개 리소스에 403 을 주면 "있는데 못 본다"로 존재를 흘린다. 일부러 **404** 로 응답하는 설계 선택이 있다(GitHub 비공개 저장소).

## 작동 원리 — 표준 예외 타입으로 매핑하면 생기는 일

```java
@ExceptionHandler(IllegalArgumentException.class)  → 404
@ExceptionHandler(IllegalStateException.class)     → 409
```

| 던지는 곳 | 의미 | 맞는 코드 | 실제 |
|---|---|---|---|
| `IAE("공유된 플랜이 아닙니다")` | 없음/숨김 | 404 | 404 (우연히 맞음) |
| `IAE("종료 시간은 시작 시간보다 빠를 수 없습니다")` | 규칙 위반 | 400 | **404** — 사용자는 "페이지를 찾을 수 없습니다"를 봄 |
| `ISE("이미 가입한 그룹입니다")` | 충돌 | 409 | 409 |
| `ISE("SHA-256 을 사용할 수 없습니다")` | **서버 버그** | 500 | **409** — 5xx 알람 안 울림, 내부 메시지 노출 |

근본 원인: **자바 표준 예외는 "의미"가 아니라 "현상"을 담고, 내 코드 밖(JDK·라이브러리)에서도 던져진다.** `NumberFormatException` 은 `IllegalArgumentException` 의 자식이라, 내 파싱 버그가 **404 + warn 로그**로 위장된다.

### 다시 설계하면

```java
// 내가 "의도해서" 던지는 실패만 이 계층으로
public abstract class BusinessException extends RuntimeException {
    public abstract HttpStatus status();
}
public class PlanNotFoundException          extends BusinessException { ... }   // 404
public class InvalidPlanTimeRangeException  extends BusinessException { ... }   // 400
public class AlreadyJoinedGroupException    extends BusinessException { ... }   // 409

@ExceptionHandler(BusinessException.class)
ResponseEntity<String> handle(BusinessException e) {
    return ResponseEntity.status(e.status()).body(e.getMessage());   // 내가 쓴 문장이라 노출 OK
}

@ExceptionHandler(Exception.class)                  // 나머지는 전부 버그로 취급
ResponseEntity<String> handleUnexpected(Exception e) {
    log.error("서버 오류", e);                                        // 예외 객체까지 → 스택 트레이스
    return ResponseEntity.internalServerError()
            .body("일시적인 오류가 발생했습니다. 잠시 후 다시 시도해 주세요.");   // 내부 정보 숨김
}
```

- 예외 이름은 **일어난 일**로 짓는다. `BadRequestException` 처럼 HTTP 코드 이름을 쓰면 여러 의미가 다시 한 이름에 섞이고 도메인이 HTTP 를 알게 된다.
- 4xx 메시지 = 사용자에게 보여 주려고 쓴 문장 → 노출 OK. 5xx 메시지 = 통제 못 한 문장(라이브러리 오류문·SQL·경로) → 숨기고 로그로.
- 비즈니스 예외를 `RuntimeException` 기반으로 두는 관례와 이어진다 ([[checked-vs-unchecked-exception]]).

## 트레이드오프 / 한계 — @ControllerAdvice 가 닿는 범위

```
브라우저 → 톰캣 → [ 필터 ] → DispatcherServlet → [ 인터셉터 ] → 컨트롤러 → 서비스
                     │              ▲
                     │              └─ 여기서 던져진 예외만 @ControllerAdvice 로 간다
                     └─ 여기서 난 예외는 DispatcherServlet 에 닿지도 않는다
```

- `@ExceptionHandler` 는 **DispatcherServlet 의 `HandlerExceptionResolver`**(`ExceptionHandlerExceptionResolver`)가 찾아 호출한다 → 컨트롤러·인터셉터 쪽 예외만.
- 시큐리티 필터의 접근 거부는 **`ExceptionTranslationFilter`** 가 처리: 익명이면 `AuthenticationEntryPoint`, 인증됐는데 권한 없으면 `AccessDeniedHandler`.
- 같은 `AccessDeniedException` 이라도 **서비스에서 던지면** `@ControllerAdvice`, **필터에서 생기면** 시큐리티가 처리한다 ([[filter-interceptor-aop]]).
- 상태 코드를 예외에 직접 두면(`status()`) 도메인이 HTTP 를 안다. 분리하려면 `ErrorCode` enum 이나 어드바이스 쪽 매핑 표로 옮긴다.

> [!WARNING]
> **오답 코너**
> - **"4xx = 서버가 거부, 5xx = 에러 발생"** → 둘 다 "실패"라 기준이 안 된다. **누구의 잘못** + **재시도 시 성공 가능성.**
> - **"400 은 사용자, 500 은 서버의 잘못"** → **4xx/5xx**. 400 은 4xx 중 Bad Request 라는 특정 코드.
> - **"시간 순서 오류는 잘못된 형식"** → 형식은 멀쩡, **규칙 위반**. 둘 다 4xx.
> - **"권한 없으면 무조건 403"** → 403 은 존재를 흘린다. **일부러 404** 로 숨기는 선택이 있다.
> - **예외 이름 `BadRequestException`** → HTTP 코드 이름은 의미를 다시 섞는다. **일어난 일**로: `InvalidPlanTimeRangeException`.
> - **5xx 응답 본문에 `e.getMessage()`** → 내부 정보 노출. 일반 문구 + `log.error(…, e)`.
> - **"@ControllerAdvice 는 인터셉터가 호출한다"** → **DispatcherServlet 의 `HandlerExceptionResolver`**. 필터 예외는 `ExceptionTranslationFilter`.

## 복습 체크

- [ ] `@ControllerAdvice` 가 없을 때의 문제 두 가지를 말할 수 있다
- [ ] 4xx 와 5xx 를 "누구의 잘못"과 "재시도"로 한 문장씩 정의할 수 있다
- [ ] 서버 버그가 4xx 로 나가면 생기는 세 가지 문제(모니터링·로그·노출)를 말할 수 있다
- [ ] 표준 예외 타입으로 매핑하면 안 되는 근본 원인을 `NumberFormatException` 예로 설명할 수 있다
- [ ] `BusinessException` 계층과 `Exception` 폴백 핸들러가 각각 무엇을 하는지(코드·로그·본문) 말할 수 있다
- [ ] 시큐리티 필터에서 막힌 요청이 `@ControllerAdvice` 로 오지 않는 이유와 대신 처리하는 주체를 말할 수 있다

## 관련

[[checked-vs-unchecked-exception]] · [[transactional-rollback-rules]] · [[filter-interceptor-aop]] · [[transactional-proxy-self-invocation]]
