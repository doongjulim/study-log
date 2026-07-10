---
date: 2026-07-10
type: raw
author: dongju
tags: [spring-events, ApplicationEventPublisher, EventListener, 이벤트기반, 모듈결합, decoupling, OCP, 응집도, cohesion, 기능별패키지, package-by-feature]
topic: Spring 이벤트로 모듈 간 결합 차단하기 (study-board 프로젝트)
summary: 기능별 패키지 구성의 근거(응집도)와, plan→notification 모듈을 인터페이스가 아닌 Spring 이벤트로 분리했을 때 결합이 어떻게 끊기는지(명령 vs 사실 통보, OCP)를 인터뷰로 검증했다.
source: session
distilled: false
---

# Spring 이벤트로 모듈 간 결합 차단하기

개인 프로젝트 study-board(Spring Boot 3.3, 게시판+플래너+알림)를 소재로 한 study-interview 세션.

## 배운 개념

### 기능별 패키지 구성 (package-by-feature)

- 레이어별(controller/service/repository) 구성은 기능이 늘수록 한 폴더가 비대해지고, 하나의 기능 수정이 세 폴더에 흩어진다.
- 기능별(post/plan/notification) 구성은 **응집도** 원칙 — "같이 바뀌는 것들은 같이 둔다". 특정 기능만 떼어 마이크로서비스로 분리할 때도 유리.

### 이벤트 발행-구독의 연결 고리 = Spring 컨테이너

```java
// plan 모듈 — NotificationService를 전혀 모른다
eventPublisher.publishEvent(new PlanSharedEvent(plan.getId(), plan.getWriter(), plan.getTitle()));
```

- `ApplicationEventPublisher`의 실체는 **`ApplicationContext` 자신**. 컨테이너가 빈 공장이면서 이벤트 중개소.
- 동작 순서:
  1. 컨테이너가 `@Component` 클래스(`NotificationEventListener`)의 **인스턴스를 빈으로** 생성
  2. 빈 초기화 후 후처리기(`EventListenerMethodProcessor`)가 `@EventListener` 붙은 **메서드를 스캔해 리스너 명단에 등록**
  3. `publishEvent()` 호출 시 **이벤트 타입 매칭**(파라미터 타입)으로 해당 메서드 호출
- 비유: 회사 게시판에 쪽지 붙이기. 발행자는 누가 읽는지 모르고, "명단 관리 + 호출"은 게시판 관리자(컨테이너)가 한다.

### 인터페이스 방식 vs 이벤트 방식

| | 인터페이스 (`NotificationSender`) | 이벤트 (`PlanSharedEvent`) |
|---|---|---|
| 발화의 성격 | "알림을 보내라" — **명령** | "일정이 공유되었다" — **사실 통보** |
| plan이 아는 것 | 후속 처리(알림 발송)가 있어야 한다는 지식 | 자기 도메인에서 일어난 사건뿐 |
| 소비자 추가 시 | plan 코드에 호출 추가 필요 | 리스너만 새로 만들면 plan 무수정 (**OCP**) |

- 확인 시나리오: "공유 시 Slack 알림도 추가" → notification(또는 신규) 모듈에 `@EventListener` 메서드 하나 추가로 끝. plan은 이미 사건을 발행 중이므로 소비자가 몇이든 존재 자체를 모른다.

## 오답·헷갈린 점

- **전**: "모듈 결합은 서비스 단에 interface를 추가해서 낮추면 된다" → **후**: 인터페이스 방식은 구현체만 숨길 뿐 "알림이 발송되어야 한다"는 후속 처리 지식이 plan에 남는다. 이벤트 방식은 사실 통보라서 그 지식 자체가 없다. (그리고 프로젝트는 이미 이벤트 방식이었다 — 내 코드가 뭘 쓰는지 먼저 확인하자.)
- **전**: "어노테이션 붙은 메서드가 빈으로 생성된다" → **후**: 빈은 **클래스의 인스턴스**. 메서드는 후처리기가 스캔해서 리스너로 **등록**되는 것. `@Autowired`/`@Transactional`과 같은 "빈 후처리" 계열 메커니즘.

## Q&A

- Q: 발행한 이벤트가 리스너까지 도달하게 연결해 주는 것은? → A: Spring 컨테이너(`ApplicationContext`). 리스너 명단을 관리하고 타입 매칭으로 호출.
- Q: Slack 알림 추가 시 plan을 안 고쳐도 되는 이유는? → A: plan은 사건만 발행하고 소비자를 모르므로, 소비자 추가는 리스너 추가로 끝난다.

## 다음 학습

- `@TransactionalEventListener` — 트랜잭션 커밋 후에만 리스너 실행하기 (지금은 `@EventListener`라 롤백돼도 알림이 나갈 수 있음)
- `@Async` 리스너 — 이벤트 처리가 발행자 스레드를 점유하는 문제
- 서버 간 이벤트: 메시지 브로커 pub/sub (SSE 노트 참고)
