---
date: 2026-07-10
type: raw
author: dongju
tags: [TransactionalEventListener, transactional-event-listener, 트랜잭션이벤트, spring-events, TransactionPhase, AFTER_COMMIT, TransactionSynchronization, 전파속성, propagation, REQUIRES_NEW, 원자성, atomicity, 최종적일관성, eventual-consistency, transactional-outbox, outbox-pattern]
topic: "@TransactionalEventListener — 트랜잭션과 이벤트의 정합성"
summary: 기본 @EventListener가 롤백돼도 알림이 나가는 정합성 문제를 출발점으로, @TransactionalEventListener의 동기화 콜백 메커니즘과 4가지 phase, AFTER_COMMIT 리스너 안 DB 쓰기가 조용히 유실되는 함정(REQUIRES_NEW로 해결), 그리고 커밋 후 실패·서버 다운에 대비하는 Transactional Outbox 패턴까지 인터뷰로 검증했다.
source: session
distilled: false
---

# @TransactionalEventListener — 트랜잭션과 이벤트의 정합성

지난 세션([2026-07-10-spring-event-module-decoupling](2026-07-10-spring-event-module-decoupling.md))의 "다음 학습" 후속. study-board의 `PlanSharedEvent` → `NotificationEventListener` 흐름을 소재로 진행.

## 배운 개념

### 존재 이유 (Pain Point)

- 기본 `@EventListener`는 `publishEvent()` 자리에서 **동기·같은 스레드·같은 트랜잭션 안**에서 즉시 실행된다.
- `publishEvent()` 다음 줄에서 예외가 터져 트랜잭션이 롤백되면: DB의 Plan은 공유 상태가 아닌데 SSE 토스트는 이미 나감 → **현실(DB)과 통보(알림)의 정합성 깨짐**.
- 같은 트랜잭션에 참여했던 `Notification` INSERT는 함께 롤백되므로 "토스트는 봤는데 목록엔 없는" 유령 알림.
- `@TransactionalEventListener` = "이 이벤트는 트랜잭션이 커밋된 뒤에만 처리해줘".

### 작동 메커니즘

- 발행 시점에 진행 중인 트랜잭션이 있으면 리스너를 즉시 호출하지 않고, **트랜잭션 동기화(TransactionSynchronization) 콜백**으로 감싸 `TransactionSynchronizationManager`에 등록해 둔다.
- 이 매니저는 **ThreadLocal 기반** — 트랜잭션이 스레드에 바인딩되어 있으므로 이벤트도 그 스레드의 동기화 목록에 매달린다.
- 방아쇠: 트랜잭션 매니저가 커밋/롤백을 완료하는 순간 등록된 콜백을 호출.

### TransactionPhase 4가지

| Phase | 시점 |
|---|---|
| `BEFORE_COMMIT` | 커밋 직전(아직 트랜잭션 안 — 실패 시 롤백 유발 가능) |
| `AFTER_COMMIT` | 커밋 성공 후 **(기본값)** |
| `AFTER_ROLLBACK` | 롤백된 후에만 (실패 보상 처리) |
| `AFTER_COMPLETION` | 커밋이든 롤백이든 종료 시 (finally 격) |

### AFTER_COMMIT의 악명 높은 함정: 조용한 유실

```java
@TransactionalEventListener  // AFTER_COMMIT
public void handlePlanShared(PlanSharedEvent event) {
    notificationService.notify(...);  // 내부에서 @Transactional + save()
}
```

- 커밋 직후엔 **끝난 트랜잭션의 리소스가 아직 스레드에 바인딩**된 상태.
- 리스너 안의 `@Transactional(REQUIRED)` 메서드는 "진행 중 트랜잭션이 있네?" 하고 **이미 커밋이 끝난 트랜잭션에 참여**한다.
- 트랜잭션 생애에서 커밋은 딱 한 번 — 끝난 트랜잭션에 커밋이 다시 오지 않으므로 INSERT는 **예외 없이 조용히 유실**된다. (비유: 이미 제출한 답안지에 몇 줄 더 적기 — 적히긴 하지만 채점자는 다시 걷어가지 않는다.)
- 해결 두 가지:
  1. `@Transactional(propagation = Propagation.REQUIRES_NEW)` — 기존 것을 suspend하고 **새 트랜잭션 + 새 커밋**
  2. `@Async` 리스너 — 새 스레드엔 끝난 트랜잭션이 바인딩되어 있지 않음

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void notify(String message, String url) { ... }
```

### 근본 한계와 Transactional Outbox 패턴

- **커밋은 되돌릴 수 없다**: 커밋 후 리스너가 실패해도 원본 트랜잭션에 영향 불가 → 원본·후속 작업의 **원자성 상실**. 후속은 재시도 등으로 **최종적 일관성**을 맞추는 설계로 전환된다.
- **서버 다운**: 커밋과 리스너 실행 사이에 서버가 죽으면 ThreadLocal의 이벤트는 증발.
- "반드시 전달" 보장이 필요하면(예: 결제 완료 통지): 원본 데이터와 **같은 DB 트랜잭션**으로 **outbox 테이블**에 이벤트 레코드를 함께 커밋(원자성 회복) → 별도 릴레이(폴러/CDC)가 outbox를 읽어 발송·재시도 = **Transactional Outbox 패턴**.

## 오답·헷갈린 점

- **전**: phase에 "플러시"가 있을 것 → **후**: 플러시는 영속성 컨텍스트의 SQL 반출 동작으로 **층위가 다름**. phase는 BEFORE_COMMIT / AFTER_COMMIT / AFTER_ROLLBACK / AFTER_COMPLETION 4개.
- **전**: "AFTER_COMMIT 리스너 안의 DB 쓰기는 커밋될 수 있다" → **후**: **조용히 유실된다.** 끝난 트랜잭션에 REQUIRED로 올라타면 커밋 기회가 영영 없음 — 예외조차 안 터지는 게 무서운 점.
- **전**: 전파 속성 이름을 "REQUIRES_SUSPEND"로 추측 → **후**: **REQUIRES_NEW.** suspend는 부수 동작이고 이름은 "새 트랜잭션 필요"에서 옴.
- **전**: 유실 방지용 이벤트 보관처로 "서버 메모리" → **후**: 서버가 죽으면 메모리도 같이 죽는다. 원본과 같은 트랜잭션으로 커밋 가능한 유일한 저장소 = **DB(outbox 테이블)**.

## Q&A

- Q: 기본 `@EventListener`는 언제/어디서 실행되나? → A: `publishEvent()` 자리에서 동기·같은 스레드·같은 트랜잭션 안에서 즉시.
- Q: `@TransactionalEventListener`는 이벤트를 어디에 보관해 뒀다 실행하나? → A: ThreadLocal 기반 `TransactionSynchronizationManager`의 동기화 콜백으로 등록, 커밋 완료가 방아쇠.
- Q: 커밋 후 리스너 실패 시 원본 커밋을 되돌릴 수 있나? → A: 불가. 원자성 상실이 이 방식의 근본 대가.
- Q: "반드시 발송" 보장은? → A: DB에 outbox 테이블 — 원본과 한 트랜잭션으로 커밋 후 릴레이가 발송·재시도.

## 다음 학습

- JPA 영속성 컨텍스트와 플러시 — "플러시는 층위가 다르다"를 제대로 이해하기 (dirty checking, 스냅샷)
- `@Async` + `@TransactionalEventListener` 조합 시 주의점 (예외 전파, 스레드 풀 설정)
- Outbox 릴레이 구현 방식 비교: 폴링 vs CDC(Debezium)
