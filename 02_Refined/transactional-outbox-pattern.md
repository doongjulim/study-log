---
type: refined
slug: transactional-outbox-pattern
tags: [transactional-outbox, outbox, 아웃박스, 발신함, 메시지유실, at-least-once, 원자성, atomicity, eventual-consistency, 최종적일관성, CDC, debezium, polling-relay, reliable-messaging]
topic: Transactional Outbox 패턴 — 이벤트의 "반드시 전달" 보장
summary: 원본 데이터와 "보낼 이벤트" 레코드를 같은 DB 트랜잭션으로 커밋해 원자성을 회복하고, 별도 릴레이가 outbox 테이블을 읽어 발송·재시도하는 신뢰성 메시징 패턴.
contributors: [dongju]
source_refs:
  - https://microservices.io/patterns/data/transactional-outbox.html
updated: 2026-07-10
---

# Transactional Outbox 패턴

## 왜 필요한가

[[transactional-event-listener]]로 커밋 후 처리를 맞춰도 남는 구멍 두 가지:

1. 커밋 후 리스너 실패 → 원본은 확정됐는데 후속 처리는 안 됨 (되돌릴 수 없음)
2. 커밋 직후·리스너 실행 직전 **서버 다운** → ThreadLocal(메모리)에 있던 이벤트 증발

즉 인메모리 이벤트는 **베스트 에포트**다. 결제 완료 통지처럼 "반드시 전달"이 필요하면 부족하다.

## 작동 원리

핵심 질문: *원본 데이터와 같은 트랜잭션으로 커밋할 수 있는 저장소는?* → **DB 자신뿐.**

1. 비즈니스 데이터(예: Plan)와 "보낼 이벤트" 레코드를 **같은 DB 트랜잭션**으로 **outbox 테이블**에 함께 커밋
   - 커밋 성공 → 둘 다 확정, 롤백 → 둘 다 취소 (**원자성 회복**)
2. 별도 **릴레이**가 outbox를 읽어 실제 발송(브로커 publish, 알림 등) 후 발송 완료 마킹
   - 릴레이 방식: **폴링**(주기적으로 미발송 행 조회) 또는 **CDC**(예: Debezium이 DB 변경 로그를 구독)
3. 발송 실패·서버 재시작 시에도 outbox에 레코드가 남아 있으므로 **재시도 가능** → at-least-once 전달 (수신 측 멱등성 필요)

```
[비즈니스 테이블]  ┐
                   ├─ 같은 트랜잭션으로 커밋
[outbox 테이블]    ┘
      │
      ▼ (폴링 or CDC)
   릴레이 ──발송·재시도──▶ 브로커/알림
```

## 트레이드오프

- 전달 시점이 즉시가 아니라 릴레이 주기만큼 지연 (최종적 일관성)
- outbox 테이블 관리(발송 완료 행 정리), 릴레이 인프라 추가
- at-least-once → 중복 발송 가능하므로 소비 측 멱등 처리 필요

## 오답 코너

> [!WARNING]
> "유실 방지를 위해 이벤트를 서버 메모리에 보관한다" (X) → 서버가 죽으면 메모리도 같이 죽는다. 원본과 한 트랜잭션으로 커밋 가능한 유일한 저장소는 **DB**다.

## 복습 체크

- [ ] 인메모리 이벤트(@TransactionalEventListener)가 못 막는 두 가지 장애 시나리오를 말할 수 있다
- [ ] "왜 하필 DB에 저장하는가"를 원자성으로 설명할 수 있다
- [ ] 릴레이의 두 가지 구현 방식(폴링/CDC)을 안다
- [ ] at-least-once와 멱등성의 관계를 설명할 수 있다

관련: [[transactional-event-listener]] · [[spring-application-events]]
