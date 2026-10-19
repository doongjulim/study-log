---
date: 2026-10-19
type: raw
author: dongju
tags: [ControllerAdvice, RestControllerAdvice, ExceptionHandler, 예외처리, exception-handling, 전역예외처리, global-exception-handler, HTTP상태코드, http-status-code, 4xx, 5xx, 400, 403, 404, 409, 422, 500, Bad-Request, Not-Found, Conflict, Unprocessable-Entity, 상태코드매핑, status-mapping, 클라이언트오류, 서버오류, 재시도, retry, 모니터링, monitoring, IllegalArgumentException, IllegalStateException, NumberFormatException, 표준예외, 커스텀예외, custom-exception, BusinessException, 예외계층, exception-hierarchy, 예외네이밍, 존재숨김, 정보노출, information-disclosure, log.error, 스택트레이스, HandlerExceptionResolver, ExceptionHandlerExceptionResolver, DispatcherServlet, ExceptionTranslationFilter, AuthenticationEntryPoint, AccessDeniedHandler, AccessDeniedException, study-board, 면접답변]
topic: "@ControllerAdvice 의 상태 코드 매핑 기준 — 자바 표준 예외 \"타입\"이 아니라 실패의 \"의미\"(누구의 잘못·재시도)로 정하고, 내가 의도한 실패만 4xx 로, 나머지는 5xx 로 (study-board GlobalExceptionHandler·ApiExceptionHandler)"
summary: 4xx 는 요청(클라이언트)의 잘못이라 고치지 않으면 재시도해도 실패하고, 5xx 는 서버의 잘못이라 나중에 성공할 수 있으며 모니터링 알람 대상이다. study-board 는 IllegalArgumentException→404, IllegalStateException→409 처럼 표준 예외 타입으로 매핑해, 규칙 위반(시간 순서·자기 응원)이 404 로, 서버 버그(SHA-256 없음)가 로그 없이 409 + 내부 메시지 노출로 나간다. 표준 예외는 의미가 아니라 현상을 담고 JDK·라이브러리도 던지기 때문이다. 의미가 드러나는 커스텀 BusinessException 계층만 4xx 로 보내고, 나머지 Exception 은 log.error + 500 + 일반 안내 문구로 처리한다. @ExceptionHandler 는 DispatcherServlet 의 HandlerExceptionResolver 가 호출하므로 필터(시큐리티)의 접근 거부는 ExceptionTranslationFilter 가 따로 처리한다.
source: session
distilled: false
---

## 배운 개념

### 1. @ControllerAdvice 가 해결한 것

```java
// 없다면: 컨트롤러 메서드마다
@PostMapping("/posts/{id}/edit")
public String edit(...) {
    try {
        postService.update(...);
        return "redirect:/posts/" + id;
    } catch (IllegalArgumentException e) {
        model.addAttribute("message", e.getMessage());
        return "error/404";
    } catch (AccessDeniedException e) {
        ...
        return "error/403";
    }
}
```

- 같은 처리가 메서드마다 **반복** → 오류 페이지 이름 하나 바꾸려면 전부 수정. try-catch 가 비즈니스 흐름을 가린다.
- 예외 처리라는 **공통 관심사를 한 곳으로** 모은 것 (같은 날 AOP 인터뷰와 같은 구조).
- study-board: HTML 오류 페이지용 `GlobalExceptionHandler`(`@ControllerAdvice`) + JSON 용 `ApiExceptionHandler`(`@RestControllerAdvice(annotations = RestController.class)`, `@Order(Ordered.HIGHEST_PRECEDENCE)` — REST 컨트롤러 예외를 먼저 잡아 JSON 으로).
- 그 외 `@ControllerAdvice` 는 예외가 아니라 `@ModelAttribute` 공통 모델 속성용(`CurrentMemberAdvice`, `SiteMetaAdvice` 등)으로도 쓰고 있음.

### 2. 상태 코드 기준: 누구의 잘못 + 재시도

| | 누구의 잘못 | 같은 요청 재시도 | 예 |
|---|---|---|---|
| **4xx** | 요청(클라이언트) | 고치지 않으면 **계속 실패** | 400 규칙 위반, 403 권한 없음, 404 없음, 409 충돌 |
| **5xx** | 서버 | 나중에 **성공할 수 있음** | 500 버그·인프라 장애 |

- 클라이언트·로드밸런서는 보통 5xx 만 재시도, 모니터링 알람도 5xx → 서버 버그가 4xx 로 나가면 **장애가 지표에서 사라진다.**
- **형식 오류 ≠ 규칙 위반.** "종료 시간 < 시작 시간"은 형식은 멀쩡하고 규칙을 어긴 것. 둘 다 4xx, 규칙 위반에 **422 Unprocessable Entity** 를 쓰는 팀도 있음.
- **존재 숨김**: "공유된 플랜이 아닙니다"는 권한 문제로 보면 403 이지만, 403 은 "있는데 못 본다"로 **존재를 흘린다.** 비공개 리소스는 일부러 404 로 응답하는 설계 선택(GitHub 비공개 저장소).

### 3. study-board 의 매핑과 문제

| 예외 | HTML (`Global`) | JSON (`Api`) |
|---|---|---|
| `MethodArgumentNotValidException` | — | 400 |
| `IllegalArgumentException` | 404 | 404 |
| `AccessDeniedException` | 403 | 403 |
| `IllegalStateException` | — (→ `Exception` 500) | 409 |
| `Exception` | 500 | — |

| 던지는 곳 | 의미 | 맞는 코드 | 실제 |
|---|---|---|---|
| `IAE("공유된 플랜이 아닙니다. id=…")` | 없음/숨김 | 404 | 404 ✅ (우연히) |
| `IAE("종료 시간은 시작 시간보다 빠를 수 없습니다.")` | 규칙 위반 | 400 | **404** — 사용자는 "페이지를 찾을 수 없습니다"를 봄 |
| `IAE("자기 자신에게는 응원을 보낼 수 없습니다.")` | 규칙 위반 | 400 | **404** |
| `ISE("이미 가입한 그룹입니다.")` | 충돌 | 409 | 409 ✅ (API) / 500 (HTML) |
| `ISE("SHA-256 을 사용할 수 없습니다.", e)` | **서버 버그** | 500 | **409** — 5xx 알람 안 울림, `ApiExceptionHandler` 는 로그도 없음, 메시지가 클라이언트에 노출 |

- 근본 원인: **자바 표준 예외는 "의미"가 아니라 "현상"을 담고, 내 코드 밖(JDK·라이브러리)에서도 던져진다.** `NumberFormatException` 은 `IllegalArgumentException` 의 자식 → 내 파싱 버그가 HTML 에선 **404 + `log.warn`** 으로 위장.
- 같은 예외가 HTML 과 JSON 에서 다른 코드(ISE: 500 vs 409)로 나가는 **불일치**도 있음.
- 참고: `DuplicateMemberException`, `LoginFailedException`, `TooManyLoginAttemptsException` 은 컨트롤러에서 catch 해 **폼을 다시 그리며** 메시지를 보여 줌(SSR 폼 검증 패턴) — 어드바이스로 가지 않음.

### 4. 다시 설계하면

```java
// 내가 "의도해서" 던지는 실패만 이 계층으로
public abstract class BusinessException extends RuntimeException {
    public abstract HttpStatus status();
}
public class PlanNotFoundException          extends BusinessException { ... }   // 404
public class InvalidPlanTimeRangeException  extends BusinessException { ... }   // 400
public class AlreadyJoinedGroupException    extends BusinessException { ... }   // 409

// ApiExceptionHandler
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

- 예외 이름은 **일어난 일**로 짓는다. `BadRequestException` 처럼 **HTTP 코드 이름**을 쓰면 여러 의미가 다시 한 이름에 섞이고(IAE 와 같은 문제), 도메인이 HTTP 를 알게 된다.
- 4xx 메시지 = 내가 사용자에게 보여 주려고 쓴 문장 → 노출 OK. 5xx 메시지 = 내가 통제 못 한 문장(라이브러리 오류문, SQL, 경로) → 숨기고 로그로.
- HTML 쪽 `GlobalExceptionHandler` 의 500 처리는 이미 이 형태(일반 문구 + `log.error`).

### 5. @ControllerAdvice 가 닿는 범위

```
브라우저 → 톰캣 → [ 필터 ] → DispatcherServlet → [ 인터셉터 ] → 컨트롤러 → 서비스
                     │              ▲
                     │              └─ 여기서 던져진 예외만 @ControllerAdvice 로 간다
                     └─ 여기서 난 예외는 DispatcherServlet 에 닿지도 않는다
```

- `@ExceptionHandler` 는 **DispatcherServlet 의 `HandlerExceptionResolver`**(`ExceptionHandlerExceptionResolver`)가 찾아 호출 → 컨트롤러·인터셉터 쪽 예외만.
- 시큐리티 필터의 접근 거부는 **`ExceptionTranslationFilter`** 가 처리: 익명이면 `AuthenticationEntryPoint`(study-board `UnauthenticatedEntryPoint` — 화면 이동은 로그인 페이지, fetch·SSE 는 401), 인증됐는데 권한 없으면 `AccessDeniedHandler`(기본 403).
- `GlobalExceptionHandler#handleAccessDenied` 가 실제로 잡는 건 **서비스에서 던진** 것 (`PostService` "본인의 게시글만 수정하거나 삭제할 수 있습니다." 등). 같은 예외 클래스라도 **어디서 던져졌느냐**로 처리 주체가 다르다.

### 면접 답변

> "상태 코드는 예외 타입이 아니라 실패의 의미로 정해야 한다고 봅니다. 클라이언트가 요청을 고쳐야 하는 실패는 4xx, 서버의 잘못은 5xx 입니다. 처음엔 IllegalArgumentException 을 404 로 매핑했는데, 그러다 보니 시간 순서 같은 규칙 위반도 404 가 됐고, 표준 예외는 라이브러리도 던져서 내 버그가 4xx 로 위장될 수 있었습니다. 그래서 의도한 실패는 의미가 드러나는 BusinessException 계층으로 만들어 4xx 로 보내고, 나머지는 전부 500 으로 응답하면서 로그를 error 로 남기고 응답에는 내부 메시지를 노출하지 않는 방식으로 바꾸는 게 맞다고 판단했습니다."

## 오답·헷갈린 점

- **전**: @ControllerAdvice 가 없을 때를 물었는데 "에러를 던져 익셉션 핸들러로 매핑" (있을 때를 답함) → **후**: 없으면 메서드마다 try-catch **반복**, 흐름이 가려짐. 공통 관심사를 한 곳으로.
- **전**: "4xx = 서버가 거부하거나 알 수 없는 경우, 5xx = 에러 발생" → **후**: 둘 다 "실패"라 기준이 안 됨. **누구의 잘못** + **재시도 시 성공 가능성.**
- **전**: "400 은 사용자의 잘못, 500 은 서버의 잘못" → **후**: **4xx/5xx**. 400 은 4xx 중 Bad Request 라는 특정 코드.
- **전**: 시간 순서 오류 = "잘못된 **형식**" → **후**: 형식은 멀쩡, **규칙 위반**. 둘 다 4xx(400 또는 422).
- **전**: "공유된 플랜이 아닙니다" = 권한 없음 → 403 → **후**: 403 은 존재를 흘린다. **일부러 404 로 숨기는** 설계 선택도 있음.
- **전**: SHA-256 오류는 "서버의 잘못, 4xx 로 나감" (결과 누락) → **후**: 409 로 나가 5xx 알람·로그·정보 노출 세 가지 문제.
- **전**: 400 예외 이름 `BadRequestException` → **후**: HTTP 코드 이름은 의미를 다시 섞는다. **일어난 일**로: `InvalidPlanTimeRangeException`. (`AlreadyJoinGroupException` 은 좋은 이름, 영어는 `AlreadyJoinedGroupException`)
- **전**: 5xx 응답 본문 = `e.getMessage()` → **후**: 내부 정보 노출. 일반 안내 문구 + 상세는 `log.error("…", e)` 로.
- **전**: "컨트롤러 어드바이스는 **인터셉터**가 호출" → **후**: **DispatcherServlet 의 `HandlerExceptionResolver`** 가 호출. 필터 예외는 **`ExceptionTranslationFilter`** 가 처리.
- ⚠️ 반복 테마: **질문 뒷부분 건너뛰기**(종료 시간 분류 질문, 모니터링 결과)가 이번에도 두 번. 그리고 **비슷한 개념의 경계**(400 vs 4xx, 형식 vs 규칙, 인터셉터 vs DispatcherServlet). 질문이 한꺼번에 많을 때 막혔음("?") → 쪼개서 하나씩 답하면 바로 풀었음. 👍 본인 코드 표를 정확히 읽고 세 IAE 를 의미로 묶어 낸 것, 의미가 드러나는 예외 이름을 바로 지은 것.

## Q&A

- **Q. @ControllerAdvice 는 왜 쓰나?** A. 컨트롤러마다 반복되는 예외 처리를 한 곳으로 모으고, 컨트롤러엔 비즈니스 흐름만 남기려고.
- **Q. 4xx 와 5xx 의 기준은?** A. 4xx 는 요청의 잘못이라 고치지 않으면 재시도해도 실패, 5xx 는 서버의 잘못이라 나중에 성공할 수 있고 모니터링 대상.
- **Q. 시작보다 이른 종료 시간은?** A. 규칙 위반이라 400(또는 422). study-board 는 IAE 라서 404 로 나간다.
- **Q. 표준 예외 타입으로 상태 코드를 매핑하면 뭐가 문제인가?** A. 한 타입에 여러 의미가 섞이고, JDK·라이브러리도 던져서 버그가 4xx 로 위장된다.
- **Q. 어떻게 고치나?** A. 의도한 실패만 의미가 드러나는 BusinessException 계층으로 4xx, 나머지 Exception 은 log.error + 500 + 일반 문구.
- **Q. 시큐리티 필터에서 막힌 요청은 @ControllerAdvice 로 오나?** A. 안 온다. @ExceptionHandler 는 DispatcherServlet 의 HandlerExceptionResolver 가 호출하는데 필터는 그 앞이다. ExceptionTranslationFilter 가 EntryPoint/AccessDeniedHandler 로 처리한다.

## 다음 학습

- study-board 에 `BusinessException` 계층 실제 적용 — IAE 19곳을 의미별로 분류해 교체, HTML/JSON 의 ISE 매핑 불일치 해소
- 상태 코드를 예외에 둘지(`status()`) vs `ErrorCode` enum 으로 분리할지, 도메인이 HTTP 를 모르게 하는 방법
- RFC 9457 `ProblemDetail`(Spring 6) 표준 오류 응답 형식
- `@ControllerAdvice` 여러 개의 우선순위 결정(`@Order`, `annotations`/`assignableTypes` 범위)과 `@Order` 를 빼면 REST 요청에 HTML 오류 뷰가 나갈 수 있는지
- SSR 폼 검증 오류를 오류 페이지가 아니라 폼 재렌더(200 + 메시지)로 처리하는 기준
- 연결: [[checked-vs-unchecked-exception]], [[transactional-rollback-rules]], [[2026-10-19-filter-interceptor-aop-auth]], [[2026-10-19-transactional-proxy-self-invocation]]
