---
type: refined
slug: singleton-bean-stateful
tags: [싱글톤, 싱글턴, singleton, 싱글톤빈, singleton-bean, 빈스코프, bean-scope, 상태, stateful, 무상태, stateless, 상태있는빈, final, 참조불변, 공유가변상태, shared-mutable-state, 스레드안전, thread-safety, 경쟁조건, race-condition, 갱신유실, lost-update, read-modify-write, 원자성, atomicity, ConcurrentHashMap, compute, AtomicLong, 멀티서버, scale-out, 수평확장, 인메모리상태, in-memory-state, SseEmitterRegistry, LoginAttemptLimiter, 로그인시도제한, 스티키세션, sticky-session, Redis, redis-pubsub, 면접답변]
topic: 싱글톤 빈이 상태를 가져도 되는가 — 상태가 필요한 빈은 같은 JVM 에선 원자적 연산으로 스레드 안전하게, 여러 서버에선 "틀려도 되는 상태인가"로 판단한다
summary: 싱글톤 빈은 컨테이너(JVM) 하나에 객체 1개를 모든 요청 스레드가 공유한다. 상태가 문제인지는 final 여부가 아니라 여러 스레드가 그 객체의 내용을 바꾸느냐로 결정된다. 로그인 실패 횟수·SSE 연결처럼 요청을 넘어 누적돼야 하는 값은 싱글톤이 오히려 필요조건이며, 이때 물을 것은 둘이다 — 같은 JVM 안에서 read-modify-write 가 원자적인가(compute, AtomicLong), 서버가 늘면 정확도만 떨어지는가 기능이 깨지는가.
contributors: [dongju]
source_refs:
  - https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html
updated: 2026-10-19
---

# 싱글톤 빈과 상태

## 개념 정의

- **싱글톤 빈** = 스프링 컨테이너(**JVM 하나**) 안에 빈 정의당 객체 **1개.** 동시 요청 100개의 스레드가 **모두 같은 객체**를 쓴다.
- 기본값인 이유: 요청마다 `new` 하면 생성·**GC 부담**이 요청 수만큼 늘어나는데, 대부분의 서비스 빈은 **무상태**라 하나를 공유해도 충분하다.
- "상태" = 요청마다 **바뀌는** 필드 값. 무엇이 상태인지는 `final` 이 결정하지 않는다:

```java
private final OrderRepository orderRepository;   // 무상태 — 안전. final 덕이 아니라 OrderRepository 에 바뀌는 값이 없어서
private final Map<String, LoginAttempts> attemptsByKey = new ConcurrentHashMap<>();   // final 이지만 상태

attemptsByKey = new HashMap<>();         // ❌ 컴파일 에러: 참조 재할당
attemptsByKey.put("1.2.3.4", attempts);  // ✅ 가능: 내용 변경
```

`final` 은 **참조**만 고정하고 내용은 막지 않는다 ([[shallow-immutability-defensive-copy]]).

## 작동 원리 — 상태가 "필요한" 싱글톤

"싱글톤 + 상태 = 나쁘다"는 절반짜리 규칙이다. 요청을 넘어 **누적·공유돼야 하는 값**은 싱글톤이어야만 동작한다.

```java
// 로그인 무차별 대입 지연 — 요청 출처(IP)별 실패 횟수
@Component
public class LoginAttemptLimiter {
    private final Map<String, LoginAttempts> attemptsByKey = new ConcurrentHashMap<>();

    public void recordFailure(String key) {
        // compute 로 읽고-고치고-쓰기를 한 번에 - 동시에 들어온 실패가 서로를 덮어쓰지 않는다
        attemptsByKey.compute(key, (k, existing) ->
                (existing != null ? existing : LoginAttempts.none()).fail(...));
    }
}

// 접속 중인 SSE 연결 — memberId → (emitterId → emitter)
@Component
public class SseEmitterRegistry {
    private final Map<Long, Map<Long, SseEmitter>> emittersByMember = new ConcurrentHashMap<>();
    private final AtomicLong sequence = new AtomicLong();
}
```

요청마다 새 객체였다면 실패 횟수는 매번 0부터 시작해 제한이 **동작하지 않는다.** 그래서 이런 빈에 던질 질문은 "상태를 가져도 되나?"가 아니라 아래 두 축이다.

### 축 1 — 같은 JVM, 여러 스레드: 원자적인가

```java
int count = attemptsByKey.getOrDefault(key, 0);  // ① 읽기
attemptsByKey.put(key, count + 1);               // ② 쓰기
// A, B 둘 다 ①에서 3 → 둘 다 4 를 씀 → 실패 1회 소실
```

- **경쟁 조건 / 갱신 유실** ([[race-condition-lost-update]]). 로그인 제한에선 공격자가 동시 요청으로 시도 횟수를 아낄 수 있다는 뜻.
- `ConcurrentHashMap` 으로 바꿔도 **해결 안 됨**: `get`·`put` 각각은 안전하지만 문제는 **둘 사이의 틈.** `compute()` 로 같은 키의 read-modify-write 를 원자화한다 ([[concurrenthashmap-vs-synchronizedmap]]).
- 카운터 `long sequence; sequence++` 도 같은 문제 → `AtomicLong.incrementAndGet()`.

### 축 2 — 여러 서버: 틀려도 되는 상태인가

싱글톤의 "1개"는 **JVM 당 1개**. 서버가 2대면 맵도 서버마다 **별개 객체**다.

| | `LoginAttemptLimiter` | `SseEmitterRegistry` |
|---|---|---|
| 서버 2대일 때 | 시도가 대수만큼 허용, 재시작 시 초기화 | 다른 서버에 연결된 회원의 알림이 **조용히 유실** |
| 성격 | **정확도 저하**, 기능은 동작 | **기능 자체가 깨짐** |
| 확장 시 해법 | 카운터를 **Redis 로 이동** | 연결은 못 옮김 → **메시지를 pub/sub 로 방송** ([[sse-server-sent-events]]) |

`SseEmitter` 는 값이 아니라 **특정 서버와 브라우저 사이의 HTTP 연결**이라 외부 저장소에 넣을 수 없다. 옮길 것은 연결이 아니라 **"회원 1에게 알림 보내 줘"라는 메시지.**

## 트레이드오프 / 한계

- 인메모리 상태는 빠르고 단순하지만 **서버 1대 · 재시작 시 소실**이 전제다. 그 전제를 코드 주석에 남겨 두면 확장할 때 옮길 자리가 보인다.
- "조용히 무시" 하는 코드(`if (emitters == null) return;`)는 멀티 서버에서 **장애 알람이 울리지 않는 장애**가 된다. 확장 시 먼저 로그·메트릭을 달 곳.
- 요청별 상태가 필요하면 싱글톤 필드가 아니라 메서드 지역 변수·파라미터로 넘기거나 request 스코프/`ThreadLocal` 을 쓴다 (스레드 풀 재사용 시 `ThreadLocal` 정리 필수).

> [!WARNING]
> **오답 코너**
> - **"싱글톤 = 빈은 상태가 변하면 안 된다"** → 그건 주의사항. 싱글톤은 **JVM 당 객체 1개를 모든 스레드가 공유**한다는 뜻.
> - **"`final` 은 필드에 담은 객체의 불변성을 보장한다"** → 반대. **참조만** 고정. `final Map` 도 `put` 가능.
> - **"`compute()` 는 간극을 해결한다"** → 이름으로 말할 것: **read-modify-write 를 원자적으로** 실행, 같은 키의 다른 스레드는 대기.
> - **"서버 B 는 A 에 로그인한 회원 정보를 몰라서 알림이 안 간다"** → 로그인 정보는 세션·토큰으로 어디서든 확인 가능. B 가 모르는 건 **SSE 연결.**

## 복습 체크

- [ ] 싱글톤 빈의 "1개"가 어떤 범위에서의 1개인지 말할 수 있다
- [ ] 싱글톤이 기본값인 이유를 말할 수 있다
- [ ] `final` 필드인데도 상태 문제가 생기는 예를 들 수 있다
- [ ] `LoginAttemptLimiter` 가 싱글톤이 아니면 무슨 일이 생기는지 말할 수 있다
- [ ] get→put 갱신 유실을 값을 따라가며 설명하고, ConcurrentHashMap 만으로 안 되는 이유와 compute 를 말할 수 있다
- [ ] 서버 2대에서 두 클래스가 깨지는 방식의 차이(정확도 vs 기능)와 각각의 해법을 말할 수 있다

## 관련

[[race-condition-lost-update]] · [[concurrenthashmap-vs-synchronizedmap]] · [[sse-server-sent-events]] · [[shallow-immutability-defensive-copy]] · [[static-util-vs-spring-bean]] · [[constructor-vs-field-injection]]
