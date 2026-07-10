---
date: 2026-07-10
type: raw
author: dongju
tags: [SSE, server-sent-events, SseEmitter, 실시간알림, realtime-notification, 폴링, polling, websocket, 비동기서블릿, async-servlet, ConcurrentHashMap, 메모리누수, memory-leak, scale-out, pub-sub, redis-pubsub]
topic: SSE(SseEmitter) 실시간 알림의 작동 원리와 한계 (study-board 프로젝트)
summary: SSE가 연결을 끊지 않고 응답을 스트림으로 흘려보내는 원리, SseEmitter return 후 스레드 반납·연결 유지 메커니즘, ConcurrentHashMap과 cleanup의 필요성, 스케일 아웃 시 JVM 로컬 한계와 pub/sub 해법을 인터뷰로 검증했다.
source: session
distilled: false
---

# SSE(SseEmitter) 실시간 알림의 작동 원리와 한계

개인 프로젝트 study-board의 SSE 알림(`SseEmitterRegistry`)을 소재로 한 study-interview 세션.

## 배운 개념

### SSE의 본질

- HTTP는 태생적으로 "클라이언트 요청 → 서버 응답 → 끝"이라 서버가 먼저 보낼 수 없다. 우회법: 폴링 / WebSocket / SSE.
- 폴링: 요청-응답-연결끊기를 반복. SSE: 브라우저가 **한 번** 요청하면 서버가 응답을 끝맺지 않는다.
- 한 문장 정의: **"연결을 끊지 않고, 응답을 스트림으로 계속 흘려보낸다"** (`Content-Type: text/event-stream`)
- 브라우저 `EventSource`는 연결이 끊기면 자동 재연결한다(이건 부가 기능이지 본질이 아님).

### SseEmitter return의 의미 (서블릿 비동기)

- 일반 컨트롤러의 return = "응답 완성, 보내고 닫아". `SseEmitter` return = "**응답 아직 안 끝남. 스레드만 돌려보내고 연결은 열어 둬**".
- subscribe를 처리한 톰캣 스레드는 return 후 **스레드 풀로 반납**되고, HTTP 연결(응답 통로)만 열린 채 남는다. 이후 `emitter.send()`는 **어느 스레드가 호출해도 된다** — 실제로 B의 공유 요청을 처리하던 스레드가 A의 emitter에 send한다.
- 비유: 식당 진동벨. 점원(스레드)은 손님 옆에서 기다리지 않고 다음 손님을 받으러 가고, 음식이 나오면 누구든 진동벨(emitter)을 울린다.

### 동시성: 왜 ConcurrentHashMap인가

```java
private final Map<Long, SseEmitter> emitters = new ConcurrentHashMap<>();
```

- `broadcast()`가 map을 **순회하는 도중**에 다른 스레드의 `add()`(새 구독)나 타임아웃 콜백의 `remove()`가 끼어들 수 있다.
- 일반 `HashMap`이면 순회 중 구조 변경으로 `ConcurrentModificationException` 또는 내부 구조 파손. `ConcurrentHashMap`은 순회 중 넣고 빼기가 안전하고, broadcast가 순회하면서 스스로 `remove(id)`하는 것까지 허용된다.

### cleanup이 없으면 생기는 두 가지 문제

```java
emitter.onCompletion(() -> emitters.remove(id));
emitter.onTimeout(() -> emitters.remove(id));
emitter.onError(e -> emitters.remove(id));
// + broadcast에서 send 실패(IOException) 시 remove
```

1. **메모리 누수** — 탭을 닫아도 map이 죽은 emitter의 참조를 쥐고 있어 GC가 수거 못 하고 무한히 쌓인다.
2. **broadcast 성능 저하** — 알림 하나에 죽은 emitter 수만 개로 send 시도 + 예외 처리 헛수고.

### 스케일 아웃의 한계와 해법

- emitter map은 **각 서버의 힙에 각자** 존재하고, Spring 이벤트도 **JVM 안에서만** 돈다.
- 서버 2대 + 로드밸런서: A는 서버1에 SSE 연결, B의 공유 요청은 서버2가 처리 → 서버2의 map은 비어 있으므로 **A는 알림을 못 받는다**.
- 해법의 본질: 이벤트를 JVM 밖 **공용 채널**로 꺼낸다 — 메시지 브로커 pub/sub(Redis Pub/Sub, Kafka 등)를 모든 서버가 구독하고, 각 서버는 자기 힙의 emitter들에게만 broadcast. (전용 푸시 서버를 두는 패턴도 결국 서버 간 전달 채널이 필요해서 본질은 같다.)
- 통찰: JVM 내 `ApplicationEventPublisher`와 서버 간 메시지 브로커는 **같은 pub/sub 원리의 스케일만 다른 버전**.

## 오답·헷갈린 점

- **전**: "SSE는 브라우저가 자동으로 연결해 주는 것" → **후**: 자동 재연결은 부가 기능. 본질은 "연결을 끊지 않고 응답을 스트림으로 흘려보내는 것".
- **전**: "cleanup이 없으면 id가 충돌하거나 잘못 삭제된다" → **후**: id는 `AtomicLong.incrementAndGet()`으로 증가만 하므로 충돌 불가. 진짜 문제는 값(죽은 emitter) 쪽 — 메모리 누수와 broadcast 성능 저하.
- **전**: "서버가 2대여도 서버2가 새 SSE 연결을 만들어 알림을 전달한다" → **후**: **서버는 브라우저에 먼저 연결할 수 없다**(그게 가능했으면 SSE가 필요 없다). 연결은 항상 브라우저가 시작하고, 각 서버의 map은 로컬이므로 다른 서버에 붙은 클라이언트에겐 전달 불가.

## Q&A

- Q: A의 emitter에 send하는 스레드는 subscribe를 받았던 스레드인가? → A: 아니다. subscribe 스레드는 반납됐고, send는 사건을 일으킨 쪽(B의 요청 처리 스레드 등) 누구든 가능.
- Q: 서버 간 알림 전달에 필요한 것은? → A: 서버들 바깥의 공용 pub/sub 채널(메시지 브로커).

## 다음 학습

- SSE vs WebSocket 트레이드오프 (단방향/양방향, 프록시 호환성, HTTP/2에서의 연결 제한)
- Redis Pub/Sub으로 다중 서버 SSE 브로드캐스트 실습
- 현재 구조가 "전체 브로드캐스트"인데 사용자별 알림으로 바꾸려면 (emitter를 userId로 키잉)
