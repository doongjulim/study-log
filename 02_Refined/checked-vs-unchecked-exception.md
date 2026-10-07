---
type: refined
slug: checked-vs-unchecked-exception
tags: [Checked-Exception, 체크예외, 확인된예외, Unchecked-Exception, 언체크예외, 비확인예외, RuntimeException, 런타임예외, Exception, Error, Throwable, 예외계층, exception-hierarchy, catch-or-declare, throws, try-catch, 예외처리, 컴파일에러, IOException, SQLException, DataAccessException, 비즈니스예외, BusinessException, 커스텀예외, custom-exception, 예외전환, exception-translation, 람다, lambda, 함수형인터페이스]
topic: Checked 와 Unchecked 예외를 가르는 기준과, 컴파일러가 Checked 에만 강제하는 것
summary: 기준은 RuntimeException 을 상속했는지 하나뿐이다(통신·비즈니스 여부와 무관). 컴파일러는 예외 발생을 검출하지 못하고, Checked 예외에 대해서만 try-catch 로 처리하거나 throws 로 떠넘기라는 catch-or-declare 규칙을 강제한다. 스프링 진영은 비즈니스 예외를 RuntimeException 기반으로 만드는 것이 관례다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-11.html
  - https://docs.oracle.com/javase/tutorial/essential/exceptions/runtime.html
updated: 2026-10-07
---

# Checked vs Unchecked 예외

## 개념 정의

**판별 기준은 상속 계층 하나뿐이다.**

```
Throwable
 ├─ Error                  (OutOfMemoryError, StackOverflowError ...)   ← Unchecked
 └─ Exception                                                           ← Checked
     ├─ IOException, SQLException ...
     └─ RuntimeException                                                ← Unchecked
         └─ IllegalArgumentException, NullPointerException ...
```

```java
class CouponNotFoundException extends Exception { }         // Checked
class CouponNotFoundException extends RuntimeException { }  // Unchecked
```

같은 쿠폰 예외라도 **무엇을 상속했느냐**만으로 갈린다. 통신이냐 비즈니스 로직이냐는 기준이 아니다.
JDK 가 `IOException` 같은 외부 자원 예외를 Checked 로 만든 건 "파일이 없거나 네트워크가 끊기는 건
**호출자가 대비할 수 있는 상황**이니 처리를 강제하자"는 **설계 의도**일 뿐, 판별 기준은 상속이다.

`Error` 는 `Exception` 의 하위가 아니다 — "예외"와 "에러"를 구분해서 말해야 한다 ([[jvm-stack-and-heap]]).

## 작동 원리 — catch or declare

컴파일러는 예외 **발생**을 미리 검출하지 못한다 (컴파일 시점엔 파일이 있는지, 쿠폰이 DB 에 있는지 모른다).
Checked 예외를 던질 수 있는 코드에 대해 **처리 방식**만 강제한다.

```java
public void readFile() {
    Files.readString(Path.of("a.txt"));   // IOException(Checked) → 컴파일 에러
}
```

1. **catch**: `try-catch` 로 여기서 직접 처리
2. **declare**: 메서드에 `throws IOException` 을 선언해 호출자에게 떠넘김

Unchecked 는 이 강제가 없다. Checked 를 `throws` 로 떠넘기면 **호출 체인 전체**에 `throws` 가 전파된다:

```java
public Coupon findCoupon(Long id) throws CouponNotFoundException {   // Checked 라면 필수
    return couponRepository.findById(id).orElseThrow(CouponNotFoundException::new);
}
```

## 트레이드오프 / 한계

| | Checked | Unchecked |
|---|---|---|
| 장점 | 처리 누락을 컴파일러가 잡아 줌 | 시그니처 오염 없음, 필요한 곳(예: `@RestControllerAdvice`)에서만 처리 |
| 단점 | `throws` 가 계층 전체로 번짐, 의미 없는 catch 남발 | 처리 누락을 컴파일러가 못 잡음 |
| `@Transactional` 기본 | **커밋** | **롤백** → [[transactional-rollback-rules]] |

- 스프링 관례: `SQLException`(Checked) 을 `DataAccessException`(Unchecked) 으로 **감싸서 다시 던진다**(예외 전환).
  비즈니스 예외도 `RuntimeException` 을 상속한 공통 부모(예: `BusinessException`)를 두는 것이 일반적이다.
- Checked 를 catch 해서 Unchecked 로 감쌀 때는 **원인(cause)을 반드시 넘긴다** — `throw new BusinessException("...", e);` 스택트레이스가 끊기지 않도록.

> [!WARNING]
> **오답 코너**
> - **"Checked 는 컴파일 과정에서 검출되는 에러"** — 컴파일러는 예외 **발생**을 검출하지 못한다. **처리 방식(catch or declare)** 을 강제할 뿐이다. 그리고 `Error` 는 별도 클래스이므로 "예외"라고 구분한다.
> - **"데이터 타입 차이 등 통신 간의 에러가 Checked"** — 예시(`IOException`, `SQLException`)에 끌려간 오해. 기준은 **`RuntimeException` 상속 여부 하나**.

## 복습 체크

- [ ] Checked 와 Unchecked 를 가르는 기준을 한 문장으로 말할 수 있는가?
- [ ] `IOException` 이 Checked 인 이유(설계 의도)와 판별 기준을 구분해서 설명할 수 있는가?
- [ ] 컴파일러가 Checked 예외에 대해 강제하는 두 가지 선택지는?
- [ ] `CouponNotFoundException extends Exception` 이라면 `orElseThrow` 를 쓰는 서비스와 컨트롤러 코드에 어떤 영향이 있는가?
- [ ] 스프링이 `SQLException` 을 `DataAccessException` 으로 바꿔 던지는 이유는?

## 관련

[[transactional-rollback-rules]] · [[jvm-stack-and-heap]] · [[java-optional]] · [[lambda-checked-exception]]
