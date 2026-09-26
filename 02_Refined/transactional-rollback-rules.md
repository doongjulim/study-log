---
type: refined
slug: transactional-rollback-rules
tags: [Transactional, "@Transactional", 트랜잭션, transaction, 롤백, rollback, 커밋, commit, 롤백규칙, rollback-rules, rollbackFor, noRollbackFor, 트랜잭션프록시, transaction-proxy, AOP, 스프링AOP, TransactionInterceptor, Checked-Exception, 체크예외, RuntimeException, 런타임예외, Error, 예외삼키기, swallowing-exception, try-catch, 원자성, atomicity, 데이터정합성, BusinessException]
topic: @Transactional 이 어떤 예외에서 롤백하는지, 그 결정을 누가 언제 내리는지
summary: 트랜잭션 프록시가 메서드 밖으로 나가는 예외의 타입을 보고 결정하며, 기본값은 RuntimeException·Error 만 롤백하고 Checked 예외는 커밋이다. 롤백 단위는 트랜잭션 전체(전부 아니면 전무)라서, 비즈니스 예외를 Checked 로 만들거나 트랜잭션 안에서 예외를 catch 로 삼키면 에러가 났는데도 전체가 커밋된다.
contributors: [dongju]
source_refs:
  - https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/rolling-back.html
  - https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-decl-explained.html
updated: 2026-09-26
---

# @Transactional 롤백 규칙

## 개념 정의

| 프록시가 본 것 | 기본 결과 |
|---|---|
| `RuntimeException` / `Error` | **rollback** |
| Checked `Exception` | **commit** |
| 예외 없음 (안에서 catch 로 삼킨 경우 포함) | commit |

Checked 예외를 커밋하는 이유: Checked 는 "호출자가 대비할 수 있는 상황"이라는 설계 의도를 따른 것 ([[checked-vs-unchecked-exception]]).

## 작동 원리 — 누가, 언제 결정하나

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

- 판단 근거는 **통과한 예외의 타입 하나**. 이후 호출자가 처리하는지는 무관하다.
- 롤백 단위는 **트랜잭션 전체** — 전부 아니면 전무. try 블록 단위가 아니다.
- 프록시를 거쳐야 하므로 같은 클래스 안의 self-invocation 에는 적용되지 않는다 ([[spring-data-repository-proxy]]).

## 트레이드오프 / 한계

### 함정 1 — Checked 비즈니스 예외는 커밋된다

```java
@Transactional
public void useCoupon(Long memberId, Long couponId) throws CouponExpiredException {
    Coupon coupon = couponRepository.findById(couponId).orElseThrow(...);
    coupon.markUsed();                                   // ① UPDATE coupon SET used = true
    pointRepository.save(new Point(memberId, 100));      // ② INSERT point
    if (coupon.isExpired()) throw new CouponExpiredException();   // ③
}
```

`CouponExpiredException extends Exception` 이면 ①② **커밋.** 사용자에겐 "만료된 쿠폰" 에러가 뜨지만
**쿠폰은 사라지고 포인트는 받은** 상태가 된다. 에러 로그는 남는데 데이터가 틀어져 있어 원인 추적이 가장 어려운 유형.

| 해결 | 장점 | 단점 |
|---|---|---|
| `@Transactional(rollbackFor = Exception.class)` | 예외 클래스를 안 바꿔도 됨 | 메서드마다 옵션 필요 — **깜빡하면 같은 버그** |
| ✅ 비즈니스 예외를 `RuntimeException` 상속 | 기본 규칙만으로 안전, 스프링 관례 | 컴파일러의 처리 강제가 사라짐 |

사람의 기억에 의존하는 방식보다 **기본값이 안전한 방식**을 고른다.

### 함정 2 — 예외를 삼키면 프록시가 못 본다

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

예외가 catch 에서 처리되어 **프록시까지 도달하지 못함** → 정상 종료로 판단 → **트랜잭션 전체 커밋** (try 안이든 밖이든).
다시 던지면 프록시가 보고 **전체 롤백** (try 밖의 ① 포함).

> 주의: 삼킨 예외가 **다른 빈의 `@Transactional` 메서드**(같은 트랜잭션에 참여, REQUIRED)에서 났다면 이야기가 다르다.
> 안쪽 프록시가 이미 rollback-only 로 표시해 두어, 바깥에서 커밋하려는 순간 `UnexpectedRollbackException` 이 난다.

> [!WARNING]
> **오답 코너**
> - **"Checked 예외면 처리가 따로 되어 있으니 경우에 따라 저장될 수 있다"** — 경우에 따라가 아니라 **기본적으로 항상 커밋**. 프록시는 예외 **타입만** 본다.
> - **"try-catch 안(또는 밖)의 내용은 롤백되지 않는다"** — 롤백 단위는 try 블록이 아니라 **트랜잭션 전체**. 삼키면 **전부 커밋**, 다시 던지면 **전부 롤백**.

## 복습 체크

- [ ] `@Transactional` 의 기본 롤백 대상과 커밋 대상 예외를 말할 수 있는가?
- [ ] 롤백 여부를 결정하는 주체와 시점을 프록시 그림으로 설명할 수 있는가?
- [ ] 쿠폰 사용 시나리오에서 예외가 Checked 일 때 생기는 데이터 버그는?
- [ ] 두 가지 해결책(`rollbackFor` vs `RuntimeException` 상속)과 어느 쪽을 고를지, 그 이유는?
- [ ] 트랜잭션 안에서 RuntimeException 을 catch 해 로그만 찍으면 어떻게 되는가? 롤백 단위는?
- [ ] 다른 빈의 트랜잭션 메서드에서 난 예외를 삼키면 왜 `UnexpectedRollbackException` 이 나는가?

## 관련

[[checked-vs-unchecked-exception]] · [[spring-data-repository-proxy]] · [[transactional-event-listener]] · [[transactional-outbox-pattern]] · [[dirty-checking]]
