---
type: refined
slug: sse-server-sent-events
tags: [SSE, server-sent-events, SseEmitter, EventSource, 실시간알림, realtime, 폴링, polling, websocket, text-event-stream, 비동기서블릿, async-servlet, ConcurrentHashMap, 메모리누수, memory-leak, scale-out, redis-pubsub, broadcast]
topic: SSE(Server-Sent Events)와 SseEmitter의 작동 원리·한계
summary: HTTP 연결을 끊지 않고 응답을 스트림으로 흘려보내 서버→클라이언트 푸시를 구현하는 기법. SseEmitter return 후 스레드 반납·연결 유지, emitter 저장소의 동시성·정리, JVM 로컬이라는 스케일 아웃 한계까지.
contributors: [dongju]
source_refs:
  - https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events
updated: 2026-07-10
---

# SSE (Server-Sent Events)

## 정의와 필요성

HTTP는 태생적으로 "클라이언트 요청 → 서버 응답 → 끝"이라 **서버가 먼저 보낼 수 없다**. 우회법이 폴링 / WebSocket / SSE.

- **폴링**: 요청-응답-연결끊기를 주기적으로 반복 — 새 소식이 없어도 왕복 낭비.
- **SSE**: 브라우저가 **한 번** 요청하면 서버가 응답을 끝맺지 않는다. 한 문장 정의 — **"연결을 끊지 않고, 응답을 스트림으로 계속 흘려보낸다"** (`Content-Type: text/event-stream`). 단방향(서버→클라이언트), 추가 프로토콜 없이 HTTP만으로 동작.
- 브라우저 `EventSource`는 연결이 끊기면 **자동 재연결**한다 — 이건 부가 기능이지 본질이 아님.

## SseEmitter return의 의미 (서블릿 비동기)

- 일반 컨트롤러의 return = "응답 완성, 보내고 닫아". `SseEmitter` return = "**응답 아직 안 끝남. 스레드만 돌려보내고 연결은 열어 둬**".
- subscribe 요청을 처리한 스레드는 return 후 **스레드 풀로 반납**되고, HTTP 연결(응답 통로)만 열린 채 남는다.
- 이후 `emitter.send()`는 **어느 스레드가 호출해도 된다** — 실제로는 사건을 일으킨 다른 사용자의 요청 처리 스레드나 스케줄러 스레드가 호출한다.
- 비유: 식당 진동벨 — 점원(스레드)은 손님 옆에서 기다리지 않고 다음 손님을 받으러 가고, 음식이 나오면 누구든 진동벨(emitter)을 울린다.

## emitter 저장소: 동시성과 정리(cleanup)

```java
private final Map<Long, SseEmitter> emitters = new ConcurrentHashMap<>();

emitter.onCompletion(() -> emitters.remove(id));
emitter.onTimeout(() -> emitters.remove(id));
emitter.onError(e -> emitters.remove(id));
// + broadcast에서 send 실패(IOException) 시에도 remove
```

- **왜 ConcurrentHashMap**: `broadcast()`가 map을 순회하는 도중 다른 스레드의 `add()`(새 구독)/`remove()`(타임아웃 콜백)가 끼어든다. 일반 HashMap이면 `ConcurrentModificationException` 또는 내부 구조 파손. ConcurrentHashMap은 순회 중 넣고 빼기, 순회하며 스스로 remove까지 안전.
- **cleanup이 없으면**: ① **메모리 누수** — 탭을 닫아도 map이 죽은 emitter 참조를 쥐고 있어 GC 불가, 무한 누적. ② **broadcast 성능 저하** — 알림 하나에 죽은 emitter 수만 개로 send 시도 + 예외 처리 헛수고.

## 스케일 아웃 한계

- emitter map은 **각 서버의 힙에 각자** 존재. 서버 2대 + LB에서 A는 서버1에 연결, 사건은 서버2에서 발생하면 → 서버2의 map엔 A가 없어 **알림 불가**. Spring 이벤트([[spring-application-events]])도 JVM 안에서만 돈다.
- 해법의 본질: 이벤트를 JVM 밖 **공용 채널**로 — 메시지 브로커 pub/sub(Redis Pub/Sub, Kafka 등)를 모든 서버가 구독하고, 각 서버는 **자기 힙의 emitter들에게만** broadcast.

## 오답 코너

> [!WARNING]
> - "SSE는 브라우저가 자동으로 연결해 주는 것" (X) → 자동 재연결은 부가 기능. 본질은 연결 유지 + 응답 스트리밍.
> - "cleanup이 없으면 id가 충돌한다" (X) → id는 `AtomicLong` 증가라 충돌 불가. 진짜 문제는 죽은 emitter **참조 누적**(메모리 누수)과 broadcast 낭비.
> - "서버가 여러 대여도 다른 서버가 새 SSE 연결을 만들어 전달한다" (X) → **서버는 브라우저에 먼저 연결할 수 없다**(그게 가능했으면 SSE가 필요 없음). 연결은 항상 브라우저가 시작한다.

## 복습 체크

- [ ] SSE를 "연결"의 관점에서 한 문장으로 정의할 수 있다 (폴링과 대비)
- [ ] SseEmitter를 return한 뒤 스레드와 연결에 각각 무슨 일이 일어나는지 설명할 수 있다
- [ ] send()를 호출하는 스레드가 subscribe 스레드가 아닌 이유를 안다
- [ ] emitter 저장소에 ConcurrentHashMap이 필요한 동시 시나리오를 들 수 있다
- [ ] cleanup 부재 시 두 가지 문제(메모리/성능)를 이름 붙여 설명할 수 있다
- [ ] 다중 서버에서 SSE가 깨지는 이유와 pub/sub 해법을 그림으로 그릴 수 있다

관련: [[spring-application-events]] · [[transactional-event-listener]]
