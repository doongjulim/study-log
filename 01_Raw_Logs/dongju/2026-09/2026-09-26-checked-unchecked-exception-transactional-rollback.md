---
date: 2026-09-26
type: raw
author: dongju
tags: [Checked Exception, 체크 예외, Unchecked Exception, 언체크 예외, RuntimeException, 런타임 예외, Exception, Error, Throwable, 예외 계층, exception hierarchy, catch or declare, throws, try-catch, 예외 처리, "@Transactional", 트랜잭션, transaction, 롤백, rollback, 커밋, commit, rollbackFor, 트랜잭션 프록시, transaction proxy, AOP, 스프링 AOP, 예외 삼키기, swallowing exception, DataAccessException, SQLException, BusinessException, 비즈니스 예외, 원자성, atomicity]
topic: Checked/Unchecked 예외를 가르는 기준과, @Transactional 이 어떤 예외에서 롤백하는지 — 그리고 그 결정을 누가 언제 내리는가
summary: Checked/Unchecked 는 RuntimeException 을 상속했는지 하나로 갈리고, 컴파일러는 Checked 에만 catch or declare 를 강제한다. @Transactional 은 메서드 밖으로 나간 예외를 프록시가 보고 판단하며, 기본값은 RuntimeException·Error 만 롤백, Checked 는 커밋이다. 그래서 비즈니스 예외를 Checked 로 만들거나 트랜잭션 안에서 예외를 catch 로 삼키면 에러가 났는데도 전체가 커밋된다.
source: session
distilled: true
distilled_at: 2026-09-26
---

## 배운 개념

### 1. 판별 기준은 상속 계층 하나 (What)

```
Throwable
 ├─ Error                  (OutOfMemoryError ...)        ← 롤백 O
 └─ Exception                                             ← Checked, 롤백 X
     ├─ IOException, SQLException ...
     └─ RuntimeException                                  ← Unchecked, 롤백 O
         └─ IllegalArgumentException, NullPointerException ...
```

```java
class CouponNotFoundException extends Exception { }         // Checked
class CouponNotFoundException extends RuntimeException { }  // Unchecked
```

- 같은 쿠폰 예외라도 **무엇을 상속했느냐**만으로 갈린다. 통신이냐 비즈니스 로직이냐는 기준이 아니다.
- JDK 가 `IOException` 같은 외부 자원 예외를 Checked 로 만든 건 "호출자가 대비할 수 있는 상황이니 처리를 강제하자"는 **설계 의도**. 의도일 뿐 판별 기준은 상속.

### 2. 컴파일러가 강제하는 것: catch or declare (Why)

```java
public void readFile() {
    Files.readString(Path.of("a.txt"));   // IOException(Checked) → 컴파일 에러
}
```

컴파일러는 예외 **발생**을 미리 검출하지 못한다(컴파일 시점엔 파일이 있는지, 쿠폰이 DB 에 있는지 모름).
Checked 예외에 대해 **처리 방식**만 강제한다:

1. `try-catch` 로 직접 처리
2. 메서드에 `throws` 를 선언해 호출자에게 떠넘김

Unchecked 는 이 강제가 없다.

`Exception` 을 직접 상속했다면 `orElseThrow(CouponNotFoundException::new)` 를 쓰는 메서드와 그 호출자 모두 `throws` 가 필요하다:

```java
public Coupon findCoupon(Long id) throws CouponNotFoundException {
    return couponRepository.findById(id)
            .orElseThrow(CouponNotFoundException::new);
}
```

### 3. @Transactional 롤백은 누가, 언제 결정하나 (How)

```
Controller
   │ useCoupon() 호출
   ▼
[트랜잭션 프록시]  ← @Transactional 이 붙으면 스프링이 끼워 넣는 대리 객체
   │  1. 트랜잭션 시작
   │  2. 진짜 useCoupon() 호출 ─────▶ CouponService.useCoupon()
   │  3. 예외가 프록시를 "통과해서" 나가는 순간
   │     → 예외 타입을 보고 commit / rollback 결정
   ▼
Controller (이후 try-catch 하든 말든 결정에 영향 없음)
```

| 프록시가 본 것 | 기본 결과 |
|---|---|
| `RuntimeException` / `Error` | rollback |
| Checked `Exception` | **commit** |
| 예외 없음 (안에서 catch 로 삼킨 경우 포함) | commit |

- 판단 기준은 **통과한 예외의 타입 하나**. 이후 컨트롤러가 처리하는지는 무관.
- 롤백 단위는 **트랜잭션 전체** — 전부 아니면 전무. try 블록 단위가 아니다.

### 4. 실제 버그 시나리오

```java
@Transactional
public void useCoupon(Long memberId, Long couponId) throws CouponExpiredException {
    Coupon coupon = couponRepository.findById(couponId).orElseThrow(...);
    coupon.markUsed();                                   // ① UPDATE coupon SET used = true
    pointRepository.save(new Point(memberId, 100));      // ② INSERT point

    if (coupon.isExpired()) {
        throw new CouponExpiredException();              // ③
    }
}
```

- `CouponExpiredException extends RuntimeException` → ①② 롤백.
- `CouponExpiredException extends Exception` → ①② **커밋.**
  만료 쿠폰인데 `used = true` 저장 + 포인트 100 지급. 사용자에겐 "만료된 쿠폰" 에러가 뜨지만 **쿠폰은 사라지고 포인트는 받은** 상태.
  에러 로그는 남는데 데이터가 틀어져 있어 운영에서 원인 찾기 가장 어려운 유형.

### 5. 해결 방법과 트레이드오프

```java
@Transactional(rollbackFor = Exception.class)   // 방법 1: Checked 도 롤백
```

| 방법 | 장점 | 단점 |
|---|---|---|
| `rollbackFor = Exception.class` | 예외 클래스를 안 바꿔도 됨 | 메서드마다 옵션 필요 — 팀원이 **깜빡하면 같은 버그** |
| ✅ 비즈니스 예외를 `RuntimeException` 상속 | 기본 규칙만으로 안전 | 컴파일러의 처리 강제가 사라짐 |

- 방법 2 선택. 에러가 났는데 데이터가 저장되는 건 매우 큰 결함이라, 사람의 기억에 의존하는 방법 1 보다 기본값으로 안전한 방법 2 가 낫다.
- 스프링 관례와도 일치: `SQLException`(Checked) → `DataAccessException`(Unchecked) 으로 감싸서 던진다. 비즈니스 예외도 `RuntimeException` 기반 공통 부모(예: `BusinessException`)를 둔다.

### 6. 예외를 삼키면 프록시가 못 본다

```java
@Transactional
public void useCoupon(Long memberId, Long couponId) {
    Coupon coupon = couponRepository.findById(couponId).orElseThrow(...);
    coupon.markUsed();                                  // ① UPDATE

    try {
        pointCalculator.calculate(memberId);            // RuntimeException 발생
    } catch (RuntimeException e) {
        log.warn("포인트 계산 실패", e);
        // throw e;  ← 이 줄이 있으면 ①까지 전부 롤백
    }
}
```

- 예외가 catch 에서 처리되어 **프록시까지 도달하지 못함** → 프록시는 정상 종료로 판단 → **트랜잭션 전체 커밋** (try 안이든 밖이든).
- `throw e;` 로 다시 던지면 프록시가 보고 **전체 롤백** (try 밖의 ① 포함).

## 오답·헷갈린 점

- **Checked 의 정의**
  - 전: "컴파일 과정에서 검출되는 에러."
  - 후: 컴파일러는 예외 **발생**을 검출하지 못한다. **처리 방식(catch or declare)** 을 강제할 뿐. 또 `Error` 는 별도 클래스이므로 "예외"로 구분해서 말한다.
- **Checked 의 기준**
  - 전: "데이터 타입 차이 등 통신 간의 에러가 Checked."
  - 후: 예시(`IOException`, `SQLException`)에 끌려간 오해. 기준은 **`RuntimeException` 상속 여부 하나**.
- **Checked 예외 시 @Transactional 결과**
  - 전: "익셉션 처리가 따로 되어 있으므로 경우에 따라 저장될 수 있다."
  - 후: 경우에 따라가 아니라 **기본적으로 항상 커밋**. 프록시는 통과한 예외의 **타입만** 보고, 이후 호출자가 처리하는지는 상관없다.
- **catch 로 삼킨 경우의 롤백 범위**
  - 전: "try-catch 안의 내용은 롤백되지 않는다" → "try-catch 밖의 내용은 롤백되지 않는다."
  - 후: 롤백 단위는 try 블록이 아니라 **트랜잭션 전체**. 예외가 프록시에 도달하지 못했으니 **안이든 밖이든 전부 커밋**. 다시 던지면 **전부 롤백**.
- **내 프로젝트의 `CouponNotFoundException`**
  - "`Exception` 을 상속했다"고 답했지만 서비스에 `throws` 가 있는지는 기억나지 않음 → 실제 코드 확인 필요 (아래 다음 학습).

## Q&A

- **Q. Checked 와 Unchecked 차이는?**
  A. `RuntimeException` 을 상속했는지로 갈린다. Checked 는 컴파일러가 try-catch 나 throws 를 강제하고, Unchecked 는 강제하지 않는다.
- **Q. @Transactional 은 어느 쪽에서 롤백하나?**
  A. 기본은 Unchecked(`RuntimeException`)와 `Error`. Checked 는 커밋한다. 결정은 메서드 밖으로 나간 예외를 트랜잭션 프록시가 보고 내린다.
- **Q. Checked 비즈니스 예외를 쓰면 어떤 버그가?**
  A. 에러 응답은 나가는데 중간 변경(쿠폰 사용 처리, 포인트 지급)이 커밋된다.
- **Q. 어떻게 막나?**
  A. `rollbackFor` 옵션 또는 `RuntimeException` 상속. 깜빡할 위험이 없는 후자를 선택.
- **Q. 트랜잭션 안에서 RuntimeException 을 catch 해서 로그만 찍으면?**
  A. 프록시가 예외를 못 보므로 전체 커밋.
- **30초 면접 답변**
  > "Checked 와 Unchecked 는 RuntimeException 을 상속했는지로 갈리고, Checked 는 컴파일러가 try-catch 나 throws 를 강제합니다. @Transactional 은 기본적으로 Unchecked 예외와 Error 에서만 롤백하고 Checked 예외는 커밋합니다. 그래서 비즈니스 예외를 Checked 로 만들면 에러가 났는데도 중간 변경이 저장되는 버그가 생길 수 있어서, 비즈니스 예외는 RuntimeException 기반으로 만들어 기본 규칙만으로 롤백되게 했습니다. 또 트랜잭션 안에서 예외를 catch 로 삼키면 프록시가 예외를 보지 못해 전체가 커밋된다는 점도 주의합니다."

## 다음 학습

- **study-board 실제 코드 확인**: `CouponNotFoundException` 등 비즈니스 예외의 상속 대상과 `throws` 여부 — Checked 라면 위 버그가 숨어 있을 수 있음
- 같은 트랜잭션의 **다른 빈** `@Transactional` 메서드에서 난 예외를 바깥에서 catch 로 삼키는 경우 — 내부 프록시가 rollback-only 로 표시해 커밋 시 `UnexpectedRollbackException` 이 나는 이유 (전파 속성 REQUIRED)
- **self-invocation**: 같은 클래스 안에서 `this.method()` 로 부르면 프록시를 안 거쳐 `@Transactional` 이 적용되지 않는 문제
- `noRollbackFor`, 전파 속성(`REQUIRES_NEW` 등)과 롤백 범위
- `@RestControllerAdvice` 로 `BusinessException` 계층을 한곳에서 HTTP 응답으로 변환하는 설계
