---
date: 2026-10-19
type: raw
author: dongju
tags: [트랜잭션 전파, transaction propagation, propagation, 전파 속성, REQUIRED, REQUIRES_NEW, NESTED, SUPPORTS, NOT_SUPPORTED, MANDATORY, NEVER, 물리 트랜잭션, 논리 트랜잭션, physical transaction, logical transaction, rollback-only, 롤백 전용, UnexpectedRollbackException, 세이브포인트, savepoint, 트랜잭션 보류, suspend, 예외 전파, try-catch, 커넥션 풀 데드락, connection pool deadlock, 데드락, deadlock, HikariCP, connectionTimeout, SQLTransientConnectionException, CannotCreateTransactionException, 풀 사이즈, pool sizing, 감사 로그, audit log, 면접 답변]
topic: 트랜잭션 전파 — 참여(REQUIRED)한 쪽의 예외는 rollback-only 로 전체를 롤백시키고(UnexpectedRollbackException), REQUIRES_NEW 는 트랜잭션만 분리할 뿐 예외는 삼키지 않으며 요청당 커넥션 2개로 풀 데드락을 부를 수 있다
summary: 전파 속성은 이미 트랜잭션이 진행 중일 때 다른 @Transactional 메서드가 참여·새로 시작·트랜잭션 없이·예외 중 무엇을 할지 고르는 설정이다. 기본값 REQUIRED 로 참여한 내부 메서드가 RuntimeException 을 던지면 프록시가 물리 트랜잭션 전체에 rollback-only 를 표시하므로, 바깥에서 catch 해도 커밋 시점에 전체 롤백 + UnexpectedRollbackException 이 난다. 독립 커밋이 필요한 로그는 REQUIRES_NEW 로 분리하되 예외는 그대로 전파되므로 바깥 try-catch 가 필수이고, 바깥 트랜잭션이 커넥션을 쥔 채 보류되어 요청당 커넥션 2개가 필요하므로 풀 크기만큼 동시 요청이 몰리면 커넥션 풀 데드락 → 30초 후 CannotCreateTransactionException 이 난다.
source: session
distilled: true
distilled_at: 2026-10-19
---

## 배운 개념

### 1. 전파 속성이란

> **이미 트랜잭션이 진행 중일 때** 다른 `@Transactional` 메서드가 호출되면, 기존 트랜잭션에 **참여할지 / 새로 만들지 / 트랜잭션 없이 실행할지 / 예외를 던질지**를 결정하는 설정.

| 속성 | 기존 트랜잭션 있음 | 없음 |
|---|---|---|
| **`REQUIRED`** (기본) | 참여 | 새로 시작 |
| **`REQUIRES_NEW`** | 보류하고 **새 물리 트랜잭션** | 새로 시작 |
| `NESTED` | 세이브포인트로 중첩 (JPA 에선 사실상 미지원) | 새로 시작 |
| `SUPPORTS` / `NOT_SUPPORTED` | 참여 / 보류하고 트랜잭션 없이 | 트랜잭션 없이 |
| `MANDATORY` / `NEVER` | 참여 / 예외 | 예외 / 트랜잭션 없이 |

- 기본 롤백 대상은 **`RuntimeException` 과 `Error`**. 체크 예외는 기본 **커밋** ([[transactional-rollback-rules]]).

### 2. 물리 트랜잭션과 논리 트랜잭션 (REQUIRED 참여)

```
물리 트랜잭션 (실제 DB 커넥션의 트랜잭션, 주인 = 처음 시작한 order() 의 프록시)
 ├─ 논리 트랜잭션 1: order()
 └─ 논리 트랜잭션 2: saveOrderLog()  ← 참여만 함. 커밋/롤백 권한 없음
```

- **모든 논리 트랜잭션이 커밋돼야** 물리 트랜잭션이 커밋, **하나라도 롤백되면** 물리 트랜잭션 롤백.
- 참여한 쪽은 주인이 아니라 직접 롤백할 수 없다 → 대신 트랜잭션 **전체**에 **rollback-only** 표시를 남긴다.

### 3. 참여한 쪽의 예외를 바깥에서 catch 하면?

```java
@Transactional
public void order(Long id) {
    orderRepository.save(new Order(id));
    try {
        orderLogService.saveOrderLog(id);   // REQUIRED (기본값)
    } catch (RuntimeException e) {
        log.warn("로그 저장 실패, 주문은 계속 진행");
    }
}
```

```
1. saveOrderLog() 의 프록시: 예외 감지 → rollback-only 표시 → 예외 다시 던짐
2. order(): try-catch 로 잡음 → 정상 종료
3. order() 의 프록시: 커밋하려는데 rollback-only → 물리 트랜잭션 전체 롤백
   + 기대(커밋)와 다른 결과이므로 UnexpectedRollbackException
```

- 주문도 롤백되고, 컨트롤러는 정상 응답이 아니라 **`UnexpectedRollbackException`** 을 받는다. "분명히 catch 했는데 왜 롤백되지?"의 정체.

### 4. REQUIRES_NEW 로 로그를 분리하려면 두 가지 모두 필요

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

| 필요한 것 | 막아주는 것 |
|---|---|
| `propagation = REQUIRES_NEW` | 로그 롤백이 주문 트랜잭션에 rollback-only 를 남기는 것 |
| 바깥 `order()` 의 **try-catch** | 로그 예외가 전파되어 주문 프록시가 롤백하는 것 |

> `REQUIRES_NEW` 는 **트랜잭션**을 분리할 뿐, **예외**를 삼켜주지 않는다. 바깥 프록시는 예외의 **출처를 구분하지 못하고** `RuntimeException` 이 도착하면 롤백한다.

### 5. REQUIRES_NEW 의 대가: 커넥션 풀 데드락

- 바깥 트랜잭션은 **보류**될 뿐 커넥션을 **반납하지 않는다** → 요청 하나가 커넥션 **2개**를 동시에 쓴다.
- 풀 10개 + 동시 주문 10개: 주문 트랜잭션이 풀을 전부 점유 → 로그 트랜잭션은 커넥션을 기다림 → 그런데 주문 트랜잭션은 로그가 끝나야 끝남 → **서로가 서로를 기다리는 데드락**.
- 타임아웃이 강제로 끊는다: HikariCP `connectionTimeout`(기본 30초) 후 `SQLTransientConnectionException`, 스프링이 **`CannotCreateTransactionException`** 으로 감싸 던짐.
- 면접 답변: "요청마다 커넥션이 2개 필요한데 풀의 10개를 주문 트랜잭션이 이미 다 점유해서, 로그 트랜잭션은 반납될 리 없는 커넥션을 기다리는 데드락에 빠지고, `connectionTimeout` 30초 후 `CannotCreateTransactionException` 이 발생합니다."
- 대응: 풀 크기를 여유 있게(HikariCP 권장 `스레드 수 × (요청당 커넥션 수 − 1) + 1`), 로그를 트랜잭션 **밖**으로 빼거나 커밋 이후 이벤트로 처리([[transactional-event-listener]]).

### 한 줄 정리

같은 날 세 인터뷰가 한 줄로 이어진다: **생성자 주입으로 받는 건 프록시**([[2026-10-19-constructor-vs-field-injection]]) → **프록시를 거쳐야 `@Transactional` 이 동작**([[2026-10-19-transactional-proxy-self-invocation]]) → **프록시가 전파 속성을 읽고 참여·분리를 결정하며, 예외만 보고 커밋/롤백을 판단**한다.

## 오답·헷갈린 점

- **전**: 전파 속성 = "트랜잭션 내에서 새로 트랜잭션을 열어 원래 트랜잭션과 상관없이 동작해야 할 때" → **후**: 그건 `REQUIRES_NEW` 의 동작. 전파는 "기존 트랜잭션이 있을 때 어떻게 할지"를 고르는 **설정 자체**이고 `REQUIRES_NEW` 는 그 **선택지 중 하나**.
- **전**: REQUIRED 참여 시 "에러 발생 시" 원래 트랜잭션까지 롤백 → **후**: 기본 롤백 대상은 **`RuntimeException`·`Error`**, 체크 예외는 기본 커밋.
- **전**: 참여한 쪽은 "롤백해야 할 **위치**를 표시한다" → **후**: 위치(세이브포인트, `NESTED` 의 방식)가 아니라 트랜잭션 **전체**에 **rollback-only** 표시.
- **전**: (결론 누락) catch 했으니 컨트롤러는 정상 응답 → **후**: 전체 롤백 + **`UnexpectedRollbackException`** 을 받는다.
- **전**: `@Transactional(type="REQUIRES_NEW")` → **후**: **`@Transactional(propagation = Propagation.REQUIRES_NEW)`**. 속성명 `propagation`, 값은 문자열이 아니라 enum.
- **전**: REQUIRES_NEW 로 분리하면 바깥 프록시는 "주문 트랜잭션을 롤백하지 않음" → **후**: 예외는 그대로 전파되고 바깥 프록시는 출처를 구분 못 해 **주문도 롤백**. try-catch 는 **지우면 안 된다**.
- **전**: 풀이 차면 로그 트랜잭션은 "앞의 트랜잭션들이 종료될 때까지 기다린다" → **후**: 앞의 트랜잭션은 **절대 종료되지 않는다** = **데드락**. 타임아웃이 끊어줄 뿐.
- **전**: "커넥션 타임에러" → **후**: `SQLTransientConnectionException`, 스프링은 `CannotCreateTransactionException` 으로 감쌈.
- ⚠️ 반복 테마: **질문의 뒷부분(결론)을 건너뜀** ("컨트롤러는 무엇을 받나", "try-catch 를 지워도 되나"), **현상에 이름을 안 붙임** ("서로 기다린다" → 데드락). 같은 날 앞 두 인터뷰와 같은 패턴이지만, 마지막 답변에서 30초·HikariCP·풀 잔여 0개를 숫자와 이름으로 말한 건 개선.

## Q&A

- **Q. 트랜잭션 전파 속성이란?** A. 이미 트랜잭션이 있을 때 다른 `@Transactional` 메서드가 참여·새로 시작·트랜잭션 없이·예외 중 무엇을 할지 고르는 설정.
- **Q. 기본값과 동작은?** A. `REQUIRED`. 있으면 참여, 없으면 새로 시작.
- **Q. 참여한 내부 메서드의 예외를 바깥에서 catch 하면 커밋되나?** A. 아니다. 내부 프록시가 rollback-only 를 표시해 전체 롤백되고 `UnexpectedRollbackException` 이 난다.
- **Q. REQUIRES_NEW 는 언제 쓰나?** A. 바깥 트랜잭션의 성패와 무관하게 독립 커밋돼야 하는 작업(감사 로그, 실패 이력).
- **Q. REQUIRES_NEW 를 쓰면 바깥 try-catch 를 지워도 되나?** A. 안 된다. 트랜잭션만 분리될 뿐 예외는 전파되어 바깥 프록시가 롤백한다.
- **Q. REQUIRES_NEW 의 대가는?** A. 요청당 커넥션 2개 → 풀 크기만큼 동시 요청 시 커넥션 풀 데드락 → `connectionTimeout` 후 `CannotCreateTransactionException`.

## 다음 학습

- `NESTED` 와 세이브포인트: `DataSourceTransactionManager` 에선 되고 JPA 에선 안 되는 이유
- `UnexpectedRollbackException` 을 피하는 다른 방법: `noRollbackFor`, 내부 메서드에서 예외를 직접 처리, `TransactionTemplate`
- 트랜잭션 보류(suspend)·재개 시 `TransactionSynchronizationManager` 가 커넥션 리소스를 어떻게 바꿔 끼우는가
- HikariCP 풀 사이징과 `leakDetectionThreshold` 로 커넥션 점유 문제 찾기
- `@TransactionalEventListener(AFTER_COMMIT)` + `REQUIRES_NEW` 조합 시 주의점
- 연결: [[transactional-rollback-rules]], [[transactional-event-listener]], [[2026-10-19-transactional-proxy-self-invocation]], [[2026-10-19-constructor-vs-field-injection]]
