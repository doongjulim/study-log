---
type: refined
slug: transactional-event-listener
tags: [TransactionalEventListener, 트랜잭션이벤트리스너, TransactionPhase, AFTER_COMMIT, BEFORE_COMMIT, AFTER_ROLLBACK, AFTER_COMPLETION, TransactionSynchronization, TransactionSynchronizationManager, 전파속성, propagation, REQUIRES_NEW, REQUIRED, 정합성, consistency]
topic: "@TransactionalEventListener — 트랜잭션 phase에 맞춘 이벤트 처리"
summary: 이벤트 리스너 실행을 트랜잭션 커밋/롤백 시점에 맞추는 메커니즘. ThreadLocal 동기화 콜백으로 지연 실행되며, AFTER_COMMIT 리스너 안의 DB 쓰기는 REQUIRES_NEW 없이는 예외 없이 조용히 유실된다.
contributors: [dongju]
source_refs:
  - https://docs.spring.io/spring-framework/reference/data-access/transaction/event.html
updated: 2026-07-10
---

# @TransactionalEventListener

## 왜 필요한가

기본 `@EventListener`는 `publishEvent()` 자리에서 **동기·같은 스레드·같은 트랜잭션 안**에서 즉시 실행된다 ([[spring-application-events]]). 그래서 발행 후 트랜잭션이 **롤백**되면:

- DB는 원상복구됐는데 알림(SSE 토스트 등)은 이미 나감 → **현실(DB)과 통보의 정합성 깨짐**
- 리스너가 같은 트랜잭션에서 저장한 행은 함께 롤백 → "토스트는 봤는데 목록엔 없는" 유령 알림

`@TransactionalEventListener` = **"이 이벤트는 트랜잭션이 커밋된 뒤에만 처리해줘."**

## 작동 원리

1. 발행 시점에 진행 중 트랜잭션이 있으면 리스너를 즉시 호출하지 않고, **트랜잭션 동기화(TransactionSynchronization) 콜백**으로 감싸 `TransactionSynchronizationManager`에 등록한다.
2. 이 매니저는 **ThreadLocal 기반** — 트랜잭션이 스레드에 바인딩되어 있으므로 이벤트도 그 스레드의 동기화 목록에 매달린다.
3. 트랜잭션 매니저가 커밋/롤백을 완료하는 순간이 방아쇠가 되어 phase에 맞는 콜백이 실행된다.

### TransactionPhase 4가지

| Phase | 시점 |
|---|---|
| `BEFORE_COMMIT` | 커밋 직전 (아직 트랜잭션 안 — 실패 시 롤백 유발 가능) |
| `AFTER_COMMIT` | 커밋 성공 후 **(기본값)** |
| `AFTER_ROLLBACK` | 롤백된 후에만 (실패 보상 처리) |
| `AFTER_COMPLETION` | 커밋이든 롤백이든 종료 시 (finally 격) |

## ⚠️ AFTER_COMMIT의 악명 높은 함정: 조용한 유실

커밋 직후엔 **끝난 트랜잭션의 리소스가 아직 스레드에 바인딩**된 상태다. 이때 리스너 안에서 `@Transactional`(기본 `REQUIRED`) DB 쓰기 메서드를 호출하면:

1. `REQUIRED`는 "진행 중 트랜잭션이 있으면 참여" → **이미 커밋이 끝난 트랜잭션에 올라탄다**
2. 트랜잭션 생애에서 커밋은 딱 한 번 — 끝난 트랜잭션에 커밋은 다시 오지 않는다
3. INSERT는 **예외 없이 조용히 유실**된다 (JPA면 플러시조차 안 일어날 수 있음)

비유: **이미 제출한 답안지**에 몇 줄 더 적기 — 적히긴 하지만 채점자는 다시 걷어가지 않는다.

해결:

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)  // 기존 것 suspend + 새 트랜잭션
public void notify(...) { ... }
```

또는 리스너에 `@Async` — 새 스레드에는 끝난 트랜잭션이 바인딩되어 있지 않다.

## 트레이드오프와 한계

- **원자성 상실**: 커밋은 되돌릴 수 없으므로, 커밋 후 리스너가 실패해도 원본에 영향 불가. "둘 다 성공 or 둘 다 실패"의 세계에서 벗어나 재시도 기반 **최종적 일관성(eventual consistency)** 설계로 전환된다.
- **유실 가능**: 커밋과 리스너 실행 사이에 서버가 죽으면 ThreadLocal의 이벤트는 증발. "반드시 전달" 보장이 필요하면 → [[transactional-outbox-pattern]].

## 오답 코너

> [!WARNING]
> - "phase에 플러시가 있다" (X) → 플러시는 영속성 컨텍스트의 SQL 반출 동작으로 **층위가 다름**. phase는 위 4개뿐.
> - "AFTER_COMMIT 리스너 안의 DB 쓰기도 커밋된다" (X) → **조용히 유실.** 예외조차 안 터지는 게 무서운 점.
> - 전파 속성 이름 "REQUIRES_SUSPEND" (X) → **REQUIRES_NEW.** suspend는 부수 동작, 이름은 "새 트랜잭션 필요"에서 옴.

## 복습 체크

- [ ] 기본 @EventListener의 정합성 문제 시나리오(롤백 + 이미 나간 알림)를 그릴 수 있다
- [ ] 이벤트가 "어디에 보관됐다가, 무엇이 방아쇠가 되어" 실행되는지 설명할 수 있다
- [ ] TransactionPhase 4가지와 각 용도를 말할 수 있다
- [ ] AFTER_COMMIT 리스너 안 DB 쓰기가 유실되는 3단계 이유를 설명할 수 있다
- [ ] REQUIRES_NEW와 @Async가 각각 왜 해법이 되는지 설명할 수 있다
- [ ] 이 방식이 치르는 대가(원자성 → 최종적 일관성)를 말할 수 있다

관련: [[spring-application-events]] · [[transactional-outbox-pattern]]
