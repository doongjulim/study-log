---
type: refined
slug: spring-application-events
tags: [spring-events, ApplicationEventPublisher, EventListener, ApplicationContext, 스프링이벤트, 이벤트기반, event-driven, 모듈결합, decoupling, 결합도, OCP, pub-sub, EventListenerMethodProcessor]
topic: Spring 애플리케이션 이벤트로 모듈 간 결합 차단
summary: ApplicationEventPublisher(실체는 ApplicationContext)로 "사실"을 발행하고 @EventListener가 구독하게 하여, 발행 모듈이 소비자를 전혀 모르게 만드는 JVM 내 pub/sub 메커니즘.
contributors: [dongju]
source_refs:
  - https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html#context-functionality-events
updated: 2026-07-10
---

# Spring 애플리케이션 이벤트로 모듈 간 결합 차단

## 정의

Spring 컨테이너가 제공하는 **JVM 내 pub/sub**. 발행 측은 `ApplicationEventPublisher.publishEvent(사건객체)`만 호출하고, 소비 측은 `@EventListener` 메서드로 구독한다. 발행자는 소비자의 존재 자체를 모른다.

```java
// plan 모듈 — notification 모듈을 전혀 모른다
eventPublisher.publishEvent(new PlanSharedEvent(planId, writer, title));

// notification 모듈
@EventListener
public void handlePlanShared(PlanSharedEvent event) { ... }
```

## 작동 원리

1. 컨테이너가 `@Component` 클래스의 **인스턴스를 빈으로** 생성한다.
2. 빈 초기화가 끝난 뒤 후처리기(`EventListenerMethodProcessor`)가 각 빈에서 `@EventListener` 메서드를 스캔해 **리스너 명단에 등록**한다.
3. `publishEvent()`가 호출되면 컨테이너가 **이벤트 타입 매칭**(리스너 파라미터 타입)으로 해당 메서드들을 호출한다.
4. `ApplicationEventPublisher`의 실체는 **`ApplicationContext` 자신** — 컨테이너가 빈 공장이면서 이벤트 중개소다.
5. 기본 `@EventListener`는 발행 지점에서 **동기·같은 스레드·같은 트랜잭션 안**에서 즉시 실행된다 (→ 트랜잭션 정합성 이슈는 [[transactional-event-listener]]).

비유: 회사 게시판에 쪽지 붙이기 — 발행자는 쪽지만 붙이고, "읽을 사람 명단 관리 + 호출"은 게시판 관리자(컨테이너)가 한다.

## 인터페이스 방식과의 비교 (핵심 통찰)

| | 인터페이스 주입 (`NotificationSender`) | 이벤트 (`PlanSharedEvent`) |
|---|---|---|
| 발화의 성격 | "알림을 보내라" — **명령** | "일정이 공유되었다" — **사실 통보** |
| 발행 모듈이 아는 것 | 후속 처리가 존재해야 한다는 지식 | 자기 도메인의 사건뿐 |
| 소비자 추가 시 | 발행 모듈 코드 수정 필요 | 리스너만 추가 — 발행 모듈 무수정 (**OCP**) |

## 한계

- **JVM 로컬**: 이벤트는 그 프로세스 안에서만 돈다. 서버가 여러 대면 다른 서버에 전달되지 않는다 → 서버 간에는 메시지 브로커 pub/sub 필요 ([[sse-server-sent-events]]의 스케일 아웃 절 참고). JVM 내 이벤트와 브로커는 **같은 pub/sub 원리의 스케일 차이**다.
- 트랜잭션 정합성: 발행 후 롤백되면 리스너가 이미 실행된 뒤다 → [[transactional-event-listener]].

## 오답 코너

> [!WARNING]
> - "어노테이션 붙은 **메서드가 빈으로** 생성된다" (X) → 빈은 **클래스의 인스턴스**. 메서드는 후처리기가 스캔해 리스너로 **등록**되는 것.
> - "모듈 결합은 인터페이스 추가로 낮추면 된다" (부분만 맞음) → 인터페이스는 구현체만 숨길 뿐, "후속 처리가 있어야 한다"는 지식이 발행 모듈에 남는다. 이벤트는 그 지식 자체를 제거한다.

## 복습 체크

- [ ] 발행한 이벤트가 리스너까지 도달하는 3단계(빈 생성 → 메서드 스캔·등록 → 타입 매칭 호출)를 설명할 수 있다
- [ ] `ApplicationEventPublisher`의 실체가 무엇인지 말할 수 있다
- [ ] "명령 vs 사실 통보" 프레임으로 인터페이스 방식과의 차이를 설명할 수 있다
- [ ] 소비자(예: Slack 알림) 추가 시 발행 모듈을 왜 안 고쳐도 되는지 OCP로 설명할 수 있다
- [ ] 기본 @EventListener가 어느 스레드·트랜잭션에서 실행되는지 안다

관련: [[package-by-feature]] · [[transactional-event-listener]] · [[transactional-outbox-pattern]]
