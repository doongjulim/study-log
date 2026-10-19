---
date: 2026-10-19
type: raw
author: dongju
tags: [싱글톤, singleton, 싱글톤 빈, singleton bean, 스프링 빈 스코프, bean scope, 상태, stateful, 무상태, stateless, final, 참조 불변, 공유 가변 상태, shared mutable state, 스레드 안전, thread safety, 경쟁 상태, race condition, 갱신 손실, lost update, read-modify-write, check-then-act, 원자성, atomicity, ConcurrentHashMap, compute, AtomicLong, 멀티 서버, scale-out, 수평 확장, 인메모리 상태, in-memory state, SseEmitter, SseEmitterRegistry, LoginAttemptLimiter, 스티키 세션, sticky session, Redis, Redis Pub/Sub, 메시지 브로커, Last-Event-ID, study-board, 면접 답변]
topic: 싱글톤 빈이 상태를 가져도 되는가 — 상태가 필요한 빈은 같은 JVM 에선 원자적 연산으로 스레드 안전하게, 여러 서버에선 "틀려도 되는 상태인가"로 판단한다 (study-board 의 SseEmitterRegistry, LoginAttemptLimiter)
summary: 싱글톤 빈은 JVM(컨테이너) 하나에 객체 1개를 모든 요청 스레드가 공유한다. 상태가 문제인지는 final 여부가 아니라 여러 스레드가 그 객체의 내용을 바꾸느냐로 결정된다(final 은 참조만 고정). 로그인 실패 횟수·SSE 연결처럼 요청을 넘어 누적돼야 하는 값은 싱글톤이 필요조건이며, get→put 사이의 틈에서 생기는 갱신 손실은 ConcurrentHashMap 만으로는 못 막고 compute·AtomicLong 같은 원자적 연산으로 막는다. 서버가 여러 대면 맵도 서버마다 따로라, LoginAttemptLimiter 는 정확도만 떨어지지만 SseEmitterRegistry 는 다른 서버에 연결된 회원의 알림이 조용히 유실되어 기능이 깨진다. SseEmitter 는 HTTP 연결이라 Redis 로 못 옮기므로 메시지를 Redis Pub/Sub 로 방송한다.
source: session
distilled: true
distilled_at: 2026-10-19
---

## 배운 개념

### 1. 싱글톤 빈이란

- 스프링 컨테이너(**JVM 하나**) 안에 빈 정의당 객체 **1개**. 동시 요청 100개의 스레드가 **모두 같은 객체**를 쓴다.
- 기본값인 이유: 요청마다 `new` 하면 생성·**GC 부담**이 요청 수만큼 늘어나는데, 대부분의 서비스 빈은 **무상태**라 하나를 공유해도 충분.

### 2. 상태가 문제인지는 `final` 이 아니라 "내용을 바꾸느냐"

```java
private final Map<String, LoginAttempts> attemptsByKey = new ConcurrentHashMap<>();

attemptsByKey = new HashMap<>();         // ❌ 컴파일 에러: 참조 재할당
attemptsByKey.put("1.2.3.4", attempts);  // ✅ 가능: 내용 변경
```

- `final` 은 **참조**를 고정할 뿐, 객체의 **내용**은 막지 않는다.
- `private final OrderRepository` 가 안전한 건 `final` 덕이 아니라 `OrderRepository` 자체가 **무상태**이기 때문. `final` 은 "참조도 안 바뀜"을 더해줄 뿐.

### 3. 상태가 "필요한" 싱글톤 — study-board 의 두 클래스

```java
// SseEmitterRegistry — 접속 중인 SSE 연결을 회원별로 관리
/** memberId → (emitterId → emitter). 같은 회원이 여러 탭으로 접속할 수 있다 */
private final Map<Long, Map<Long, SseEmitter>> emittersByMember = new ConcurrentHashMap<>();
private final AtomicLong sequence = new AtomicLong();

// LoginAttemptLimiter — 요청 출처(IP)별 로그인 실패 횟수로 무차별 대입을 늦춤
private final Map<String, LoginAttempts> attemptsByKey = new ConcurrentHashMap<>();

public void recordFailure(String key) {
    LocalDateTime now = LocalDateTime.now(clock);
    // compute 로 읽고-고치고-쓰기를 한 번에 - 동시에 들어온 실패가 서로를 덮어쓰지 않는다
    LoginAttempts updated = attemptsByKey.compute(key, (ignored, existing) ->
            (existing != null ? existing : LoginAttempts.none())
                    .fail(now, THRESHOLD, WINDOW, BLOCK));
    ...
}
```

- 로그인 실패 횟수, 접속 중인 연결은 **요청을 넘어 누적·공유돼야 하는 값** → 싱글톤은 문제가 아니라 **필요조건**. 요청마다 새 객체면 실패 횟수가 매번 0 → 제한이 동작하지 않는다.
- 그래서 "싱글톤인데 상태를 가져도 되나?"의 답: **스레드 안전하게 만들었고, 서버 확장 시의 한계를 안다.**

### 4. 같은 JVM, 여러 스레드: 경쟁 상태와 원자성

```java
// 만약 이렇게 짰다면
int count = attemptsByKey.getOrDefault(key, 0);  // ① 읽기
attemptsByKey.put(key, count + 1);               // ② 쓰기
```

```
현재 카운트 3, 같은 IP 실패 요청 2개 동시 도착
스레드 A: ① 3 읽음
스레드 B: ① 3 읽음
스레드 A: ② 4 씀
스레드 B: ② 4 씀   → 최종 4 (5여야 함). 실패 1회 소실
```

- **경쟁 상태(Race Condition)**, 그중 한쪽 쓰기가 다른 쪽을 덮어써 사라지는 **갱신 손실(Lost Update)** ([[race-condition-lost-update]]). 로그인 제한에선 공격자가 동시 요청으로 시도 횟수를 아낄 수 있다는 뜻.
- `ConcurrentHashMap` 으로만 바꿔도 **해결 안 됨**: `get()`·`put()` **각각**은 안전하지만 문제는 **둘 사이의 틈** ([[concurrenthashmap-vs-synchronizedmap]]).
- 해법은 **read-modify-write 를 원자적으로**:
  - `ConcurrentHashMap.compute()` — 같은 키에 대해 람다 전체가 원자적. 같은 키의 다른 스레드는 대기.
  - `AtomicLong.incrementAndGet()` — `long sequence; sequence++;` 였다면 emitterId 가 중복될 수 있었다.

### 5. 여러 서버: 인메모리 상태의 한계

```
         로드밸런서
        ┌────┴────┐
     서버 A      서버 B
 emittersByMember  emittersByMember     ← 서로 다른 객체 (싱글톤 = JVM 당 1개)
   (회원 1 연결)      (비어 있음)
```

```java
// 서버 B 에서 send(1L, ...) 실행
Map<Long, SseEmitter> emitters = emittersByMember.get(memberId);  // null
if (emitters == null) {
    return;   // "접속 중이 아니면 조용히 무시한다" → 예외도 로그도 없이 알림 유실
}
```

| | `LoginAttemptLimiter` | `SseEmitterRegistry` |
|---|---|---|
| 서버 2대일 때 | 시도 횟수가 대수만큼 허용, 재시작 시 초기화 | 다른 서버에 연결된 회원 알림이 **조용히 유실** (약 절반) |
| 성격 | **정확도 저하**, 기능은 동작 | **기능 자체가 깨짐** |
| 확장 시 해법 | 카운터를 **Redis 로 이동** (코드 주석에 이미 명시) | 연결은 못 옮김 → **메시지를 Redis Pub/Sub 로 방송** |

- `SseEmitter` 는 값이 아니라 **특정 서버와 브라우저 사이의 HTTP 연결**을 붙잡은 객체 → 외부 저장소에 넣을 수 없다. 옮길 건 **연결**이 아니라 **"회원 1에게 알림 보내줘"라는 메시지**. 모든 서버가 구독하고, 각자 자기 맵에 있으면 보내고 없으면 무시.
- **스티키 세션은 무효**: 한 **사용자**의 요청을 같은 서버로 묶을 뿐인데, 알림은 보내는 사람(회원 2)과 받는 사람(회원 1)이 다르다.
- study-board 의 안전망: 알림 DB 저장 + `eventId` 를 실어 보내고 재연결 시 `Last-Event-ID` 이후를 `sendTo()` 로 재전송 → 실시간 전송을 놓쳐도 **재연결하면 복구** ([[sse-server-sent-events]]).

### 한 줄 정리

"싱글톤 + 상태 = 나쁘다"는 절반짜리 규칙. 상태가 필요한 빈이라면 두 가지를 묻는다: **같은 JVM 안에서 스레드 안전한가?**(원자성) / **서버가 늘면 틀려도 되는 상태인가?**(정확도 저하 vs 기능 파손).

## 오답·헷갈린 점

- **전**: 싱글톤 = "하나의 빈은 상태가 변하면 안 된다" → **후**: 그건 싱글톤일 때의 **주의사항**. 싱글톤은 **JVM 당 객체 1개를 모든 스레드가 공유**한다는 뜻.
- **전**: (누락) 싱글톤이 기본값인 이유 → **후**: 요청마다 생성하면 생성·GC 부담. 대부분 무상태라 공유로 충분.
- **전**: `final` 은 "선언 시 필드에 담은 객체의 불변성을 보장" → **후**: 반대. `final` 은 **참조만** 고정, **내용**은 막지 않는다. `final Map` 도 `put` 가능.
- **전**: 동시 수정 문제를 메커니즘으로만 설명 (이름 없음) → **후**: **경쟁 상태(Race Condition)**, **갱신 손실(Lost Update)**.
- **전**: `compute()` 는 "다른 스레드 사이에서 같은 값을 수정하는 상황에 대한 간극 해결" → **후**: **read-modify-write 를 원자적으로** 실행. 같은 키의 다른 스레드는 대기.
- **전**: 서버 B 가 모르는 건 "A 에 로그인되어 있는 회원정보" → **후**: **SSE 연결(`SseEmitter`)**. 로그인 정보는 세션·토큰으로 어디서든 확인 가능.
- **전**: 멀티 서버에서 깨지는 이유 = "로드밸런서가 한번 연결되면 해당 서버로 보내서" → **후**: 그건 스티키 세션이고, 보내는 사람 ≠ 받는 사람이라 해결책도 못 된다. 진짜 이유는 맵이 서버마다 **별개 객체**.
- ⚠️ **회상 갭**: [[race-condition-lost-update]], [[concurrenthashmap-vs-synchronizedmap]] 는 이미 02_Refined 에 있는 개념인데 **본인 코드 맥락**에서 용어가 바로 나오지 않았다 → 복습 대상.
- ⚠️ 반복 테마: **두 질문 중 힌트 쪽에만 답하고 결론을 건너뜀** ("다른 객체이다"), **현상에 이름을 안 붙임** ("간극 해결"). 같은 날 앞 세 인터뷰와 같은 패턴. 답하기 전에 "질문이 몇 개였지?" 세기. 👍 개선점: 경쟁 상태를 "둘 다 3을 읽고 각자 +1 해서 4"처럼 **값을 따라가며** 설명.

## Q&A

- **Q. 싱글톤 빈이란?** A. 컨테이너(JVM) 하나에 빈 정의당 객체 1개, 모든 요청 스레드가 공유.
- **Q. `final` 필드면 상태 문제가 없나?** A. 아니다. `final` 은 참조만 고정. 내용을 여러 스레드가 바꾸면 문제.
- **Q. `LoginAttemptLimiter` 가 싱글톤이 아니면?** A. 요청마다 실패 횟수가 0부터 시작해 제한이 동작하지 않는다. 싱글톤이 필요조건.
- **Q. `HashMap` 에 get → +1 → put 이면?** A. 동시 요청이 같은 값을 읽어 갱신 손실(경쟁 상태).
- **Q. `ConcurrentHashMap` 으로만 바꾸면?** A. 여전히 갱신 손실. get·put 사이의 틈이 문제. `compute()` 로 원자적으로.
- **Q. 서버 2대에서 `SseEmitterRegistry` 는?** A. 맵이 서버마다 따로라 다른 서버에 연결된 회원 알림이 조용히 유실. 연결은 못 옮기므로 Redis Pub/Sub 로 메시지를 방송.
- **Q. `LoginAttemptLimiter` 는 서버 2대여도 괜찮나?** A. 시도 허용 횟수가 대수만큼 늘고 재시작 시 초기화되지만, 목적(무차별 대입 지연)은 달성. 정확한 공유 카운터가 필요하면 Redis.

## 다음 학습

- `SseEmitterRegistry` 에서 회원의 안쪽 맵이 비어도 `emittersByMember` 에서 제거되지 않는 점 — 메모리 증가 범위와 `computeIfPresent` 로 정리할 때의 동시성 주의점
- Redis Pub/Sub 의 at-most-once 전달 특성과 Redis Streams·Kafka 와의 차이 (구독자가 잠깐 끊기면?)
- 프로토타입·request 스코프 빈을 싱글톤에 주입할 때의 문제 (`ObjectProvider`, scoped proxy)
- `ThreadLocal` — 싱글톤 안에서 요청별 상태를 두는 다른 방법과 스레드 풀 재사용 시 누수
- 연결: [[race-condition-lost-update]], [[concurrenthashmap-vs-synchronizedmap]], [[sse-server-sent-events]], [[static-util-vs-spring-bean]], [[2026-10-09-static-shared-mutable-state]], [[2026-10-19-constructor-vs-field-injection]]
