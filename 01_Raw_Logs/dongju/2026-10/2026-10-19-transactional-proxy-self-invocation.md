---
date: 2026-10-19
type: raw
author: dongju
tags: [Transactional, 트랜잭션, transaction, AOP, 관점 지향 프로그래밍, aspect-oriented programming, 공통 관심사, cross-cutting concern, 프록시, proxy, CGLIB, 다형성, polymorphism, 업캐스팅, 동적 디스패치, dynamic dispatch, PlatformTransactionManager, setAutoCommit, 자동 커밋, autocommit, 커넥션 풀, connection pool, HikariCP, connectionTimeout, SQLTransientConnectionException, self-invocation, 자기 호출, 내부 호출, this 호출, 프록시 우회, REQUIRES_NEW, 전파, propagation, 트랜잭션 범위, 외부 API, 별도 빈 분리, 면접 답변]
topic: "@Transactional 은 AOP 프록시가 원본 호출 전후로 트랜잭션을 여닫는 것 — 같은 클래스 내부 호출(this)은 프록시를 우회해 어노테이션이 무시되고, 정석 해법은 별도 빈으로 분리해 주입받는 것"
summary: 스프링은 원본 대신 CGLIB 프록시(원본 상속·오버라이드)를 빈으로 등록하고, 프록시가 PlatformTransactionManager 로 커넥션 획득·setAutoCommit(false)·commit/rollback·풀 반환을 원본 호출 전후에 수행한다. 같은 클래스 안의 this.saveOrder() 는 원본에서 직접 호출되므로 프록시를 거치지 않아 @Transactional 이 무시된다. 바깥 메서드에 @Transactional 을 다는 건 범위만 넓히고(외부 API 대기 중 커넥션 점유 → 풀 고갈) 내부 메서드의 REQUIRES_NEW 같은 속성은 여전히 무시하므로, 별도 빈으로 분리해 생성자 주입받아 프록시를 거치게 해야 한다.
source: session
distilled: true
distilled_at: 2026-10-19
---

## 배운 개념

### 1. @Transactional 이 없애준 반복 코드

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
    con.close();                // 풀에서 빌린 커넥션 → 실제 연결 종료가 아니라 풀에 반환
}
```

### 2. 동작 원리: AOP + 프록시

- 트랜잭션·로깅·보안처럼 여러 클래스에 흩어진 **공통 관심사**를 비즈니스 로직 밖으로 분리하는 기법이 **AOP**(관점 지향 프로그래밍). 스프링은 이를 **프록시**로 구현한다.
- 스프링은 원본 대신 **프록시를 빈으로 등록**한다. 프록시는 원본 호출 전후로 위 JDBC 흐름을 수행한다 (실제로는 `PlatformTransactionManager` 의 `getTransaction()` / `commit()` / `rollback()` 에 위임).

```java
// 개념적으로 단순화한 프록시
public class OrderServiceProxy extends OrderService {
    private final OrderService target;  // 진짜 OrderService

    @Override
    public void order(Long id) {
        // (A) 커넥션 획득 + setAutoCommit(false) — 트랜잭션 시작
        try {
            target.order(id);           // 비즈니스 로직만 있는 원본
            // (B) commit
        } catch (RuntimeException e) {
            // (C) rollback
            throw e;
        } finally {
            // (D) 커넥션 풀에 반환
        }
    }
}
```

- Spring Boot 기본은 **CGLIB**: 원본 클래스를 **상속**하고 메서드를 **오버라이드**한 프록시.
  - 상속 → 프록시를 `OrderService` 타입 변수에 담을 수 있다 (**업캐스팅**)
  - 오버라이드 → `orderService.order()` 호출 시 런타임에 프록시의 메서드가 실행된다 (**동적 디스패치**)
  - 둘을 묶은 게 **다형성**. 그래서 호출자(컨트롤러)는 프록시인지 모른다.

### 3. self-invocation: 같은 클래스 내부 호출

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
  └─ proxy.order(1L)          ← order()엔 @Transactional 없음 → 트랜잭션 없이 위임
       └─ target.order(1L)
            └─ this.saveOrder(1L)   ← this = 원본. 프록시 우회 → 어노테이션을 읽어줄 주체가 없음
```

> `@Transactional` 은 어노테이션이 스스로 일하는 게 아니라, **프록시가 호출을 가로챌 때만** 의미가 있다.

### 4. 오답 해법: 바깥 메서드에 @Transactional

```java
@Transactional
public void order(Long id) {
    saveOrderLog(id);              // 내부 호출 — REQUIRES_NEW 무시
    externalPaymentApi.call(id);   // 외부 결제 API, 응답 3초
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveOrderLog(Long id) { ... }   // 결제가 실패해도 로그는 남아야 함
```

- 내부 메서드는 자기 어노테이션이 아니라 바깥 트랜잭션에 **얹혀 갈 뿐**.
- **범위 확대**: 외부 API 를 기다리는 3초 동안 커넥션을 **점유한 채 풀에 반납하지 않음** → 트래픽이 몰리면 풀 고갈 → 다른 요청은 `connectionTimeout`(HikariCP 기본 30초) 대기 후 **`SQLTransientConnectionException`**. 느린 API 하나가 **DB 를 쓰는 모든 요청을 멈추는** 장애로 번진다.
- **속성 무시**: `REQUIRES_NEW` 를 읽을 프록시가 없으니 로그도 `order()` 트랜잭션에 얹혀 결제 실패 시 **함께 롤백** → 로그 유실.

### 5. 정석 해법: 별도 빈으로 분리 + 주입

```java
@Service
public class OrderLogService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveOrderLog(Long id) { ... }
}

@Service
public class OrderService {
    private final OrderLogService orderLogService;   // 주입되는 건 원본이 아니라 프록시

    public OrderService(OrderLogService orderLogService) {
        this.orderLogService = orderLogService;
    }

    @Transactional
    public void order(Long id) {
        orderLogService.saveOrderLog(id);   // 다른 빈의 프록시 경유
        externalPaymentApi.call(id);
    }
}
```

- 결제 API 예외 시: **주문은 롤백, 로그는 유지**.
- 이유: `OrderLogService` 의 **프록시가** `REQUIRES_NEW` 를 읽고 기존 트랜잭션을 **보류**한 뒤 **새 트랜잭션**(새 커넥션)을 열어 로그를 저장하고 `saveOrderLog()` 종료 시점에 **즉시 커밋**한다. 결제 API 호출 전에 이미 커밋돼 있어 이후 롤백의 영향을 받지 않는다.
- 참고: `REQUIRES_NEW` 실행 중에도 바깥 트랜잭션은 커넥션을 **계속 쥐고 있어서** 요청 하나가 커넥션 **2개**를 동시에 쓴다. 풀이 작으면 이것만으로도 고갈 가능.

| 방법 | 결과 | 문제 |
|---|---|---|
| 바깥 `order()` 에 `@Transactional` | 내부 메서드가 바깥 트랜잭션에 얹혀 감 | 범위 확대, 내부 속성(`REQUIRES_NEW` 등) 무시 |
| **별도 빈 분리 + 주입** ✅ | 다른 빈의 프록시 경유 → 속성 정상 동작 | 클래스 증가 (대신 책임도 분리) |

### 한 줄 정리

같은 날 첫 인터뷰([[2026-10-19-constructor-vs-field-injection]])와 연결: **생성자 주입으로 받는 건 프록시**이고, **다른 빈을 주입받아 호출해야** 프록시를 거친다. 그래서 self-invocation 의 정석 해법이 "별도 빈 + 주입".

## 오답·헷갈린 점

- **전**: JDBC 트랜잭션 = "try-catch 로 묶어 예외 시 롤백" → **후**: `setAutoCommit(false)` 로 **시작**, 성공 시 `commit`, 실패 시 `rollback`, `finally` 에서 **풀에 반환**까지가 한 세트.
- **전**: 커넥션을 "종료한다" → **후**: 풀에서 빌린 커넥션의 `close()` 는 **풀에 반환**.
- **전**: 공통 관심사를 분리하는 기법 = "SOLID 기법" → **후**: **AOP**. SOLID 는 클래스 설계 원칙 5가지.
- **전**: 프록시의 원본 호출 전 처리 = "DB 커넥션 연결" → **후**: 연결만으로는 자동 커밋 모드. **커넥션 획득 + `setAutoCommit(false)`** 가 트랜잭션 시작.
- **전**: 프록시가 부모 타입 변수에서 동작하는 이유 = "상속(override)" → **후**: 상속(업캐스팅) + 오버라이드(동적 디스패치) = **다형성**.
- **전**: self-invocation 이유 = "타겟임으로 동작하지 않는다" (한 줄) → **후**: `this` = 원본이라 **프록시를 우회** → 어노테이션을 해석할 주체가 없다. 호출 흐름으로 설명할 것.
- **전**: 해결책 = "`order()` 에 `@Transactional` 을 단다" → **후**: 범위만 넓어지고 내부 메서드 속성은 **여전히 무시**. 원인(프록시 우회)을 고친 게 아니다.
- **전**: "`saveOrderLog()` 를 `order()` 안에서 **주입받는다**" → **후**: 주입 단위는 **빈(객체)**. 메서드를 **별도 빈으로 분리**해 그 빈을 주입받는다.
- **전**: `REQUIRES_NEW` 는 "옵션을 읽고 새로운 **프록시**를 열어" → **후**: 프록시는 빈 생성 시 이미 하나. 호출 시 새로 열리는 건 **트랜잭션**(새 커넥션).
- **전**: "커넥션풀 익셉션" → **후**: 풀 고갈 시 `connectionTimeout` 대기 후 **`SQLTransientConnectionException`**.
- ⚠️ 반복 테마: 방향은 맞는데 **한 단어로 끝내거나 이름 대신 현상을 말함** (같은 날 첫 인터뷰의 "에러가 발생한다" 3회, 10/18 "용어로 환원하지 못함"과 같은 패턴). 면접에선 **무엇이 → 왜 → 그래서 어떻게 된다**를 한 문장에.

## Q&A

- **Q. @Transactional 은 어떻게 동작하나?** A. AOP 로 원본 대신 프록시를 빈으로 등록하고, 프록시가 원본 호출 전후로 `PlatformTransactionManager` 를 통해 트랜잭션 시작·커밋·롤백을 수행한다.
- **Q. 컨트롤러는 왜 프록시인지 모르나?** A. CGLIB 프록시는 원본을 상속·오버라이드하므로 다형성에 의해 부모 타입으로 주입·호출된다.
- **Q. 같은 클래스에서 `@Transactional` 메서드를 부르면?** A. `this` 는 원본이라 프록시를 우회 → 트랜잭션이 적용되지 않는다.
- **Q. 바깥 메서드에 `@Transactional` 을 달면?** A. 내부 메서드는 얹혀 갈 뿐 속성은 무시. 외부 API 가 끼면 커넥션 점유 → 풀 고갈 → `SQLTransientConnectionException`.
- **Q. 정석 해법?** A. 별도 빈으로 분리해 생성자 주입받아 호출 → 그 빈의 프록시를 거친다.
- **Q. 분리 후 결제 실패 시 주문·로그는?** A. `OrderLogService` 프록시가 `REQUIRES_NEW` 로 새 트랜잭션을 열어 즉시 커밋하므로, 주문은 롤백되고 로그는 남는다.

## 다음 학습

- `private` / `final` 메서드에 `@Transactional` 을 달면? (CGLIB 오버라이드 불가 관점)
- JDK 동적 프록시(인터페이스 기반) vs CGLIB, Spring Boot 가 CGLIB 를 기본으로 바꾼 이유
- self-invocation 의 다른 해법: `TransactionTemplate`(프로그래밍 방식), 자기 자신 주입, `AopContext.currentProxy()` 의 트레이드오프
- 트랜잭션과 커넥션을 스레드에 묶는 `TransactionSynchronizationManager`(ThreadLocal) — 리포지토리가 프록시가 연 커넥션을 어떻게 찾는가
- 연결: [[transactional-rollback-rules]], [[spring-data-repository-proxy]], [[2026-10-19-constructor-vs-field-injection]]
