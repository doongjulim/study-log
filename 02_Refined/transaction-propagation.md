---
type: refined
slug: transaction-propagation
tags: [트랜잭션전파, 전파속성, transaction-propagation, propagation, Propagation, REQUIRED, REQUIRES_NEW, NESTED, SUPPORTS, NOT_SUPPORTED, MANDATORY, NEVER, 물리트랜잭션, 논리트랜잭션, physical-transaction, logical-transaction, rollback-only, 롤백전용, setRollbackOnly, UnexpectedRollbackException, 세이브포인트, savepoint, 트랜잭션보류, suspend, 예외전파, try-catch, 감사로그, audit-log, 커넥션풀데드락, connection-pool-deadlock, 데드락, deadlock, HikariCP, connectionTimeout, SQLTransientConnectionException, CannotCreateTransactionException, 풀사이즈, pool-sizing, 면접답변]
topic: 트랜잭션 전파 — REQUIRED 로 참여한 쪽의 예외는 rollback-only 로 전체를 롤백시키고, REQUIRES_NEW 는 트랜잭션만 분리할 뿐 예외는 삼키지 않으며 요청당 커넥션 2개로 풀 데드락을 부를 수 있다
summary: 전파 속성은 이미 트랜잭션이 진행 중일 때 다른 @Transactional 메서드가 참여·새로 시작·트랜잭션 없이·예외 중 무엇을 할지 고르는 설정이다. 기본값 REQUIRED 로 참여한 내부 메서드가 RuntimeException 을 던지면 그 프록시가 물리 트랜잭션 전체에 rollback-only 를 표시하므로, 바깥에서 catch 해도 커밋 시점에 전체 롤백 + UnexpectedRollbackException 이 난다. 독립 커밋이 필요한 작업은 REQUIRES_NEW 로 분리하되 예외는 그대로 전파되므로 바깥 try-catch 가 필수이고, 바깥 트랜잭션이 커넥션을 쥔 채 보류되어 요청당 커넥션 2개가 필요하므로 풀 크기만큼 동시 요청이 몰리면 커넥션 풀 데드락이 난다.
contributors: [dongju]
source_refs:
  - https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html
  - https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing
updated: 2026-10-19
---

# 트랜잭션 전파 (Propagation)

## 개념 정의

> **이미 트랜잭션이 진행 중일 때** 다른 `@Transactional` 메서드가 호출되면, 기존 트랜잭션에 **참여할지 / 새로 만들지 / 트랜잭션 없이 실행할지 / 예외를 던질지** 를 결정하는 설정.

트랜잭션이 하나도 없을 때는 거의 의미가 없다. `REQUIRED`, `REQUIRES_NEW` 는 이 질문에 대한 **각각의 답(선택지)** 이다.

| 속성 | 기존 트랜잭션 있음 | 없음 |
|---|---|---|
| **`REQUIRED`** (기본) | 참여 | 새로 시작 |
| **`REQUIRES_NEW`** | 기존을 보류하고 **새 물리 트랜잭션** | 새로 시작 |
| `NESTED` | 세이브포인트로 중첩 (JPA 에선 사실상 미지원) | 새로 시작 |
| `SUPPORTS` / `NOT_SUPPORTED` | 참여 / 보류하고 트랜잭션 없이 | 트랜잭션 없이 |
| `MANDATORY` / `NEVER` | 참여 / 예외 | 예외 / 트랜잭션 없이 |

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)   // 속성명 propagation, 값은 enum
```

전파는 **프록시가** 읽는다. 같은 클래스 내부 호출이면 전파 속성도 무시된다 ([[transactional-proxy-self-invocation]]).

## 작동 원리

### 물리 트랜잭션과 논리 트랜잭션 (REQUIRED)

```
물리 트랜잭션 (실제 DB 커넥션의 트랜잭션, 주인 = 처음 시작한 order() 의 프록시)
 ├─ 논리 트랜잭션 1: order()
 └─ 논리 트랜잭션 2: saveOrderLog()  ← 참여만 함. 커밋/롤백 권한 없음
```

- **모든 논리 트랜잭션이 커밋돼야** 물리 트랜잭션이 커밋, **하나라도 롤백되면** 물리 트랜잭션 롤백.
- 참여한 쪽은 주인이 아니라 직접 롤백할 수 없다 → 트랜잭션 **전체**에 **rollback-only** 표시를 남긴다.

### 참여한 쪽의 예외를 바깥에서 catch 하면

```java
@Transactional
public void order(Long id) {
    orderRepository.save(new Order(id));
    try {
        orderLogService.saveOrderLog(id);   // REQUIRED (다른 빈)
    } catch (RuntimeException e) {
        log.warn("로그 저장 실패, 주문은 계속 진행");
    }
}
```

```
1. saveOrderLog() 의 프록시: RuntimeException 감지 → rollback-only 표시 → 예외 다시 던짐
2. order(): try-catch 로 잡음 → 정상 종료
3. order() 의 프록시: 커밋하려는데 rollback-only → 물리 트랜잭션 전체 롤백
   + 기대(커밋)와 다르므로 UnexpectedRollbackException
```

주문도 롤백되고, 컨트롤러는 정상 응답이 아니라 **`UnexpectedRollbackException`** 을 받는다. "분명히 catch 했는데 왜 롤백되지?" 의 정체. (예외가 프록시를 거치지 않은 일반 메서드에서 났다면 삼킨 순간 커밋된다 — [[transactional-rollback-rules]])

### REQUIRES_NEW 로 독립 커밋하기 — 두 가지가 모두 필요

```java
@Service
public class OrderLogService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveOrderLog(Long id) { ... }
}
```

```
order() 의 프록시: 주문 트랜잭션 시작
  └─ order(): orderRepository.save(...)
       └─ saveOrderLog() 의 프록시: 주문 트랜잭션 보류 → 로그 트랜잭션 새로 시작
            └─ saveOrderLog(): RuntimeException 발생!
          saveOrderLog() 의 프록시: 로그 트랜잭션 롤백 → 예외를 다시 던짐
       order(): try-catch 가 없다면 → 예외가 그대로 위로 전파
order() 의 프록시: RuntimeException 도착 → 주문 트랜잭션도 롤백
```

| 필요한 것 | 막아 주는 것 |
|---|---|
| `REQUIRES_NEW` | 로그 롤백이 주문 트랜잭션에 rollback-only 를 남기는 것 |
| 바깥 `order()` 의 **try-catch** | 로그 예외가 전파되어 주문 프록시가 롤백하는 것 |

> `REQUIRES_NEW` 는 **트랜잭션**을 분리할 뿐, **예외**를 삼켜 주지 않는다. 바깥 프록시는 예외의 **출처를 구분하지 못하고** `RuntimeException` 이 도착하면 롤백한다.

반대로 로그가 성공하면 `saveOrderLog()` 종료 시점에 **즉시 커밋**되므로, 그 뒤 주문이 롤백돼도 로그는 남는다. 감사 로그·실패 이력처럼 **바깥 성패와 무관하게 남아야 하는 작업**이 용도.

## 트레이드오프 / 한계 — 커넥션 풀 데드락

- 바깥 트랜잭션은 **보류**될 뿐 커넥션을 **반납하지 않는다** → 요청 하나가 커넥션 **2개**를 동시에 쓴다.
- 풀 10개 + 동시 주문 10개: 주문 트랜잭션이 풀을 전부 점유 → 로그 트랜잭션은 커넥션을 기다림 → 주문 트랜잭션은 로그가 끝나야 끝남 → **서로가 서로를 기다리는 데드락.**
- 타임아웃이 강제로 끊는다: HikariCP `connectionTimeout`(기본 30초) 후 `SQLTransientConnectionException`, 스프링이 **`CannotCreateTransactionException`** 으로 감싸 던진다.
- 면접 답변: "요청마다 커넥션이 2개 필요한데 풀의 10개를 주문 트랜잭션이 이미 다 점유해서, 로그 트랜잭션은 반납될 리 없는 커넥션을 기다리는 데드락에 빠지고, `connectionTimeout` 30초 후 `CannotCreateTransactionException` 이 발생합니다."
- 대응: 풀 크기를 여유 있게(HikariCP 권장 `스레드 수 × (요청당 커넥션 수 − 1) + 1`), 로그를 트랜잭션 **밖**으로 빼거나 커밋 이후 이벤트로 처리 ([[transactional-event-listener]]).

> [!WARNING]
> **오답 코너**
> - **"전파 속성 = 트랜잭션 안에서 새 트랜잭션을 열어 독립적으로 동작하는 것"** → 그건 `REQUIRES_NEW` 의 동작. 전파는 "기존 트랜잭션이 있을 때 어떻게 할지"를 고르는 **설정 자체**.
> - **"참여한 쪽은 롤백할 위치를 표시한다"** → 위치(세이브포인트, `NESTED` 방식)가 아니라 트랜잭션 **전체**에 **rollback-only.**
> - **"예외를 catch 했으니 주문은 커밋된다"** → 전체 롤백 + `UnexpectedRollbackException.`
> - **`@Transactional(type="REQUIRES_NEW")`** → `@Transactional(propagation = Propagation.REQUIRES_NEW)`. 값은 문자열이 아니라 enum.
> - **"REQUIRES_NEW 로 분리하면 바깥은 롤백하지 않는다"** → 예외는 전파되고 바깥 프록시는 출처를 구분 못 한다. **try-catch 는 지우면 안 된다.**
> - **"로그 트랜잭션은 앞의 트랜잭션들이 끝날 때까지 기다린다"** → 앞의 트랜잭션은 **절대 끝나지 않는다** = **데드락.** "커넥션 타임에러" → `SQLTransientConnectionException` / `CannotCreateTransactionException`.

## 복습 체크

- [ ] 전파 속성이 "어떤 상황에서 무엇을" 결정하는 설정인지 한 문장으로 말할 수 있다
- [ ] REQUIRED 와 REQUIRES_NEW 의 동작을 트랜잭션 유무별로 말할 수 있다
- [ ] 물리/논리 트랜잭션과 rollback-only 를 그림으로 설명할 수 있다
- [ ] 참여한 내부 메서드 예외를 바깥에서 catch 하면 결과와 예외 이름을 말할 수 있다
- [ ] REQUIRES_NEW 로 로그를 살리려면 왜 try-catch 도 필요한지 설명할 수 있다
- [ ] REQUIRES_NEW 의 커넥션 풀 데드락 시나리오를 숫자(풀 10, 요청 10)로 설명하고 대응책을 말할 수 있다

## 관련

[[transactional-proxy-self-invocation]] · [[transactional-rollback-rules]] · [[transactional-event-listener]] · [[transactional-outbox-pattern]] · [[open-in-view-osiv]]
