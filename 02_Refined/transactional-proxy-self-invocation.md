---
type: refined
slug: transactional-proxy-self-invocation
tags: [Transactional, "@Transactional", 트랜잭션, transaction, 트랜잭션동작원리, AOP, 관점지향프로그래밍, aspect-oriented-programming, 공통관심사, cross-cutting-concern, 프록시, proxy, 트랜잭션프록시, CGLIB, JDK동적프록시, 다형성, polymorphism, 업캐스팅, 동적디스패치, dynamic-dispatch, PlatformTransactionManager, setAutoCommit, 자동커밋, autocommit, 커넥션풀반환, self-invocation, 자기호출, 내부호출, this호출, 프록시우회, 별도빈분리, 트랜잭션범위, 외부API, 커넥션점유, 면접답변]
topic: "@Transactional 은 AOP 프록시가 원본 호출 전후로 트랜잭션을 여닫는 것 — 같은 클래스 내부 호출(this)은 프록시를 우회해 어노테이션이 무시되고, 정석 해법은 별도 빈으로 분리해 주입받는 것"
summary: 스프링은 원본 대신 CGLIB 프록시(원본 상속·오버라이드)를 빈으로 등록하고, 프록시가 PlatformTransactionManager 로 커넥션 획득·setAutoCommit(false)·commit/rollback·풀 반환을 원본 호출 전후에 수행한다. 같은 클래스 안의 this 호출은 프록시를 거치지 않아 @Transactional 이 무시된다. 바깥 메서드에 @Transactional 을 다는 건 범위만 넓히고 내부 메서드의 전파 속성은 여전히 무시하므로, 별도 빈으로 분리해 주입받아 프록시를 거치게 해야 한다.
contributors: [dongju]
source_refs:
  - https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-decl-explained.html
  - https://docs.spring.io/spring-framework/reference/core/aop/proxying.html
updated: 2026-10-19
---

# @Transactional 의 프록시 동작과 self-invocation

## 개념 정의

`@Transactional` 이 없앤 반복 코드:

```java
Connection con = dataSource.getConnection();
try {
    con.setAutoCommit(false);   // 트랜잭션 시작 (기본은 SQL 한 줄마다 자동 커밋)
    // 비즈니스 로직 + SQL
    con.commit();
} catch (Exception e) {
    con.rollback();
    throw e;
} finally {
    con.close();                // 풀에서 빌린 커넥션 → 연결 종료가 아니라 풀에 반환
}
```

트랜잭션·로깅·보안처럼 여러 클래스에 흩어진 **공통 관심사**를 비즈니스 로직 밖으로 분리하는 기법이 **AOP** 이고, 스프링은 이를 **프록시**로 구현한다. 스프링은 원본 대신 **프록시를 빈으로 등록**한다.

## 작동 원리

### 프록시가 하는 일

```java
// 개념적으로 단순화. 실제로는 PlatformTransactionManager 에 getTransaction/commit/rollback 위임
public class OrderServiceProxy extends OrderService {
    private final OrderService target;   // 진짜 OrderService

    @Override
    public void order(Long id) {
        // 커넥션 획득 + setAutoCommit(false)
        try {
            target.order(id);            // 비즈니스 로직만 있는 원본
            // commit
        } catch (RuntimeException e) {
            // rollback (롤백 규칙: [[transactional-rollback-rules]])
            throw e;
        } finally {
            // 커넥션 풀에 반환
        }
    }
}
```

### 호출자가 프록시인지 모르는 이유 — CGLIB 과 다형성

Spring Boot 기본은 **CGLIB**: 원본 클래스를 **상속**하고 메서드를 **오버라이드**한 프록시를 만든다.

- 상속 → 프록시를 `OrderService` 타입 변수에 담을 수 있다 (**업캐스팅**)
- 오버라이드 → `orderService.order()` 호출 시 런타임에 프록시의 메서드가 실행된다 (**동적 디스패치**)
- 둘을 묶은 게 **다형성**. 생성자 주입으로 받는 `OrderService` 도 사실 프록시다 ([[constructor-vs-field-injection]]).

### self-invocation — 같은 클래스 내부 호출

```java
@Service
public class OrderService {
    public void order(Long id) {          // @Transactional 없음
        saveOrder(id);                    // = this.saveOrder(id)
    }

    @Transactional
    public void saveOrder(Long id) { ... }
}
```

```
Controller
  └─ proxy.order(1L)                ← order() 엔 @Transactional 없음 → 트랜잭션 없이 위임
       └─ target.order(1L)
            └─ this.saveOrder(1L)   ← this = 원본. 프록시 우회 → 어노테이션을 해석할 주체가 없음
```

> `@Transactional` 은 어노테이션이 스스로 일하는 게 아니라 **프록시가 호출을 가로챌 때만** 의미가 있다.

## 트레이드오프 / 한계 — 해결책 비교

### ❌ 바깥 메서드에 `@Transactional` 을 단다

```java
@Transactional
public void order(Long id) {
    saveOrderLog(id);              // 내부 호출 — REQUIRES_NEW 무시
    externalPaymentApi.call(id);   // 외부 결제 API, 응답 3초
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveOrderLog(Long id) { ... }   // 결제가 실패해도 로그는 남아야 함
```

- 내부 메서드는 자기 어노테이션이 아니라 바깥 트랜잭션에 **얹혀 갈 뿐**. `REQUIRES_NEW` 를 읽을 프록시가 없으니 결제 실패 시 로그도 **함께 롤백**.
- **범위 확대**: 외부 API 를 기다리는 동안 커넥션을 **점유한 채 풀에 반납하지 않는다** → 트래픽이 몰리면 풀 고갈 → 다른 요청은 HikariCP `connectionTimeout`(기본 30초) 대기 후 `SQLTransientConnectionException`. 느린 API 하나가 **DB 를 쓰는 모든 요청을 멈춘다.**

### ✅ 별도 빈으로 분리해 주입받는다

```java
@Service
public class OrderLogService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveOrderLog(Long id) { ... }
}

@Service
public class OrderService {
    private final OrderLogService orderLogService;   // 주입되는 건 프록시

    public OrderService(OrderLogService orderLogService) {
        this.orderLogService = orderLogService;
    }

    @Transactional
    public void order(Long id) {
        try {
            orderLogService.saveOrderLog(id);   // 다른 빈의 프록시 경유 → 새 트랜잭션, 즉시 커밋
        } catch (RuntimeException e) { ... }     // 예외 전파 차단 — 이유는 [[transaction-propagation]]
        externalPaymentApi.call(id);             // 예외 → 주문만 롤백, 로그는 유지
    }
}
```

| 방법 | 결과 | 문제 |
|---|---|---|
| 바깥에 `@Transactional` | 내부가 바깥 트랜잭션에 얹혀 감 | 범위 확대, 내부 속성 무시 |
| **별도 빈 분리 + 주입** ✅ | 다른 빈의 프록시 경유 → 속성 정상 동작 | 클래스 증가 (대신 책임도 분리) |

그 외 해법(`TransactionTemplate` 프로그래밍 방식, 자기 자신 주입, `AopContext.currentProxy()`)도 있지만 원인(프록시 우회)을 구조로 없애는 건 분리다.

> [!WARNING]
> **오답 코너**
> - **"JDBC 트랜잭션 = try-catch 로 묶어 예외 시 롤백"** → `setAutoCommit(false)` 로 **시작**, `commit`, `rollback`, `finally` 에서 **풀에 반환**까지 한 세트.
> - **"공통 관심사 분리 기법 = SOLID"** → **AOP.** SOLID 는 클래스 설계 원칙.
> - **"프록시는 원본 호출 전에 커넥션을 연결한다"** → 연결만으로는 자동 커밋 모드. `setAutoCommit(false)` 가 트랜잭션 시작.
> - **"프록시가 부모 타입으로 동작하는 이유 = 상속"** → 상속(업캐스팅) + 오버라이드(동적 디스패치) = **다형성.**
> - **"내부 호출 문제는 바깥에 `@Transactional` 을 달면 해결"** → 범위만 넓어지고 내부 속성은 **여전히 무시**. 원인을 고친 게 아니다.
> - **"메서드를 주입받는다"** → 주입 단위는 **빈(객체).** 메서드를 별도 빈으로 분리해 그 빈을 주입받는다.
> - **"REQUIRES_NEW 는 새로운 프록시를 연다"** → 프록시는 빈 생성 시 이미 하나. 호출 시 새로 열리는 건 **트랜잭션**(새 커넥션).

## 복습 체크

- [ ] `@Transactional` 이 없앤 JDBC 반복 코드 4단계를 순서대로 쓸 수 있다
- [ ] AOP 와 프록시의 관계, 프록시가 원본 호출 전후에 하는 일을 말할 수 있다
- [ ] CGLIB 프록시를 호출자가 모르는 이유를 다형성(업캐스팅+동적 디스패치)으로 설명할 수 있다
- [ ] self-invocation 에서 트랜잭션이 안 걸리는 이유를 호출 흐름(this = 원본)으로 그릴 수 있다
- [ ] 바깥 메서드에 `@Transactional` 을 다는 해법의 두 가지 문제를 말할 수 있다
- [ ] 별도 빈 분리가 정석인 이유를 "프록시 경유"로 설명할 수 있다

## 관련

[[transactional-rollback-rules]] · [[transaction-propagation]] · [[spring-data-repository-proxy]] · [[constructor-vs-field-injection]] · [[open-in-view-osiv]] · [[filter-interceptor-aop]]
