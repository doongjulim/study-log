---
type: refined
slug: concurrenthashmap-vs-synchronizedmap
tags: [ConcurrentHashMap, 컨커런트해시맵, synchronizedMap, Collections.synchronizedMap, Hashtable, synchronized, 락, lock, 락범위, lock-granularity, 칸단위락, bin-lock, CAS, compare-and-swap, volatile, 락없는읽기, lock-free-read, ConcurrentModificationException, fail-fast, modCount, 약한일관성, weakly-consistent, 순회중수정, 싱글턴빈공유상태, shared-mutable-state, SseEmitter, 로그인시도제한, login-attempt-limiter, 복합연산, compound-operation, 원자적연산, atomic-operation, compute, computeIfAbsent, merge, putIfAbsent, get-then-put, read-modify-write, 갱신유실, lost-update, Recursive-update, null금지, 리사이즈, resize, help-transfer, Redis, 면접답변]
topic: ConcurrentHashMap 을 쓰는 이유와 synchronizedMap 과의 차이 — 락 범위(맵 전체 vs 칸), 락 없는 읽기, 순회 중 수정, 그리고 get→put 이 아닌 compute 로 복합 연산을 원자화
summary: 싱글턴 빈의 공유 맵은 추가·삭제·순회가 서로 다른 스레드에서 동시에 일어난다. HashMap 은 동시 쓰기에서 엔트리를 잃고 순회 중 수정에 ConcurrentModificationException 을 던진다. synchronizedMap 은 맵 전체에 락 하나라 순회를 지키려면 맵 전체를 잠근 채 작업해야 하고, 그 안에서 네트워크 I/O 를 하면 모든 요청이 멈춘다. ConcurrentHashMap(Java 8+)은 get 에 락이 없고, 빈 칸은 CAS, 노드가 있는 칸은 첫 노드에만 synchronized 를 걸며, 순회는 약한 일관성이다. 단 get 후 put 은 여전히 원자적이지 않아 갱신 유실이 생기므로 compute/computeIfAbsent 로 읽고-고치고-쓰기를 한 번에 한다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Collections.html#synchronizedMap(java.util.Map)
updated: 2026-10-11
---

# ConcurrentHashMap vs synchronizedMap

## 개념 정의 — 왜 필요한가

싱글턴 빈(`@Component`)의 필드는 힙에 하나뿐이고 모든 요청 스레드가 공유한다 ([[jvm-stack-and-heap]]). 실시간 알림 연결 저장소를 예로 들면:

```java
@Component
public class SseEmitterRegistry {
    private final Map<Long, Map<Long, SseEmitter>> emittersByMember = new ConcurrentHashMap<>();

    public SseEmitter add(Long memberId) {
        ...
        Map<Long, SseEmitter> emitters =
                emittersByMember.computeIfAbsent(memberId, key -> new ConcurrentHashMap<>());
        emitters.put(emitterId, emitter);
        emitter.onTimeout(() -> emitters.remove(emitterId));
        ...
    }

    public void send(Long memberId, ...) {
        Map<Long, SseEmitter> emitters = emittersByMember.get(memberId);
        if (emitters == null) return;
        emitters.forEach((emitterId, emitter) -> {
            try { emitter.send(...); }
            catch (IOException | IllegalStateException e) {
                emitters.remove(emitterId);   // 순회 도중 스스로 삭제
            }
        });
    }
}
```

| 상황 | 메서드 | 맵에 하는 일 |
|---|---|---|
| 새 탭이 알림 구독 (요청 스레드) | `add` → `computeIfAbsent`, `put` | 쓰기 (추가) |
| 예전 탭 30분 타임아웃 (컨테이너 스레드) | `onTimeout` → `remove` | 쓰기 (삭제) |
| 댓글 알림 발송 (다른 요청 스레드) | `send` → `forEach`, 실패 시 `remove` | 읽기(순회) + 가끔 쓰기 |

### HashMap 이면

- 동시 쓰기 → 엔트리가 조용히 사라짐 ([[race-condition-lost-update]]).
- 순회 중 구조 변경 → `modCount` 불일치 → **`ConcurrentModificationException`** (fail-fast 이터레이터, 최선 노력일 뿐 보장은 아님).
- 위 `send()` 는 순회 도중 **스스로** `remove` → HashMap 이면 **다른 스레드 없이도** CME.

## 작동 원리

### synchronizedMap 의 한계

```java
Map<Long, SseEmitter> emitters = Collections.synchronizedMap(new HashMap<>());
```

- `get`/`put`/`remove` **메서드 하나하나**를 맵 전체 락으로 감싼다 → 각 호출은 안전.
- **순회는 못 지킨다.** `forEach` 는 "다음 꺼내기" 반복이고, 락은 호출 사이마다 풀린다 (check-then-act 와 같은 구조).
- 공식 문서대로 `synchronized (map) { 순회 }` 로 감싸면 그 안의 `emitter.send()` = **네트워크 전송** 동안 맵 전체가 잠긴다 → 느린 클라이언트 하나 때문에 **모든 회원**의 구독·타임아웃 정리·알림이 대기.
- **락 비용 = 범위(맵 전체) × 쥐는 시간(I/O)**.

### ConcurrentHashMap (Java 8+)

[[hashmap-internals]] 의 `Node[] table` + 칸별 체인 구조 위에서:

| 동작 | `synchronizedMap` | `ConcurrentHashMap` |
|---|---|---|
| `get` | 맵 전체 락 | **락 없음** (volatile 읽기) |
| 빈 칸에 `put` | 맵 전체 락 | **CAS** 로 노드 끼워 넣기 |
| 노드가 있는 칸에 쓰기 | 맵 전체 락 | **그 칸의 첫 노드에만** `synchronized` |
| 순회 | 직접 `synchronized(map)`, 아니면 CME | 락 없이 **약한 일관성** — 순회 중 추가·삭제 허용 |
| 기다리는 경우 | 어떤 두 연산이든 (먼저 락 잡은 쪽 우선, 순서 보장 없음) | **같은 칸에 쓰기**끼리, 또는 **리사이즈 중**(쓰려는 스레드가 옮기기를 도움) |

- CAS: "지금 값이 내가 본 값 그대로면 새 값으로, 아니면 실패" 를 CPU 명령 하나로 원자적으로. 실패하면 재시도.
- 비유: synchronizedMap = **맵 전체가 줄 하나**, ConcurrentHashMap = **칸마다 각자의 줄**, 읽기는 줄을 서지 않는다.

### 복합 연산은 여전히 원자적이지 않다

```java
// ❌ 각 호출은 안전하지만 사이에 틈이 있다
LoginAttempts existing = attemptsByKey.get(key);                           // ① 읽기
LoginAttempts updated  = (existing != null ? existing : none()).fail(...);  // ② 고치기
attemptsByKey.put(key, updated);                                            // ③ 쓰기
```

같은 IP 의 로그인 실패 2건이 동시에, 현재 3회:

```
         스레드 A                                 스레드 B
 t1   ① get → 3
 t2                                             ① get → 3      ← A가 고치기 전 값을 읽음
 t3   ② fail → 4
 t4                                             ② fail → 4
 t5   ③ put → 맵의 값 = 4
 t6                                             ③ put → 맵의 값 = 4   ← A의 결과를 덮어씀
```

최종 **4**, 기대값 **5** → **갱신 유실**(read-modify-write). 공격자가 실패를 **동시에 몰아** 보내면 차단 임계값 도달이 늦어진다 ([[offline-vs-online-attack]]).

```java
// ✅ 람다 실행 동안 그 키의 칸 락을 쥔다 → 읽고-고치고-쓰기가 원자적
LoginAttempts updated = attemptsByKey.compute(key, (ignored, existing) ->
        (existing != null ? existing : LoginAttempts.none()).fail(now, THRESHOLD, WINDOW, BLOCK));
```

- 원자적 복합 연산 도구: `compute`, `computeIfAbsent`, `computeIfPresent`, `merge`, `putIfAbsent`.
- 위 `add()` 의 `computeIfAbsent` 도 같은 이유 — `get` → `put` 으로 나누면 두 탭이 동시에 첫 구독할 때 내부 맵이 두 번 만들어져 한쪽 연결이 사라질 수 있다.

> "스레드 안전한 메서드" 를 조합한다고 스레드 안전해지지 않는다.

## 트레이드오프 / 한계

- **`compute` 람다는 짧게.** 도는 동안 그 칸이 잠긴다. 람다 안에서 I/O 금지, **같은 맵 재수정 금지** (`IllegalStateException: Recursive update` 가능).
- **null 금지.** 키·값 모두 `null` 불가 — `get` 이 `null` 이면 "없음" 이 확실하도록.
- **`size()`·순회는 근사치.** 약한 일관성이라 순회 중 변경이 보일 수도, 안 보일 수도 있다.
- **서버 한 대 기준.** 메모리 맵이라 서버가 여러 대면 대수만큼 시도가 허용되고 재시작하면 초기화된다. 정확한 공유 카운터가 필요하면 Redis 등으로 옮긴다. SSE 연결도 서버별로 흩어진다 ([[sse-server-sent-events]]).

### 면접 답변 뼈대

> "실시간 알림의 SSE 연결 저장소와 로그인 시도 제한 카운터에 ConcurrentHashMap 을 썼습니다. 둘 다 싱글턴 빈의 필드라 여러 요청 스레드가 동시에 추가, 삭제, 순회를 합니다.
> `synchronizedMap` 은 맵 전체에 락 하나라서, 알림을 보내려고 순회하는 동안 맵 전체를 잠근 채 네트워크 전송을 하게 됩니다. 느린 클라이언트 하나 때문에 모든 회원의 구독과 알림이 멈출 수 있죠. ConcurrentHashMap 은 읽기에 락이 없고 쓰기는 칸 단위로만 잠그며, 순회 중 수정도 예외 없이 허용합니다.
> 다만 ConcurrentHashMap 이어도 `get` 후 `put` 은 원자적이지 않아서, 로그인 실패 카운트는 `compute` 로 읽고-고치고-쓰기를 한 번에 처리해 동시 실패가 서로를 덮어쓰지 않게 했습니다."

> [!WARNING]
> **오답 코너**
> - **"순회 중 수정하면 에러가 난다"** — 이름까지: **`ConcurrentModificationException`** (fail-fast, `modCount`). ConcurrentHashMap 은 약한 일관성이라 나지 않는다.
> - **"synchronizedMap 이면 순회도 안전하다"** — 메서드 단위 락은 호출 사이마다 풀린다. 순회는 직접 `synchronized(map)` 으로 감싸야 하고, 그러면 맵 전체가 잠긴다.
> - **"ConcurrentHashMap 에서도 읽기·쓰기가 동시에 일어나면 기다린다"** — `get` 은 **락이 없다.** 기다리는 건 같은 칸 쓰기끼리, 리사이즈 중뿐이다.
> - **"ConcurrentHashMap 이니 get 후 put 도 안전하다"** — 각 호출만 안전하다. 사이 틈에서 갱신 유실 → `compute`.
> - **"synchronizedMap 에선 먼저 들어온 등록이 끝난 뒤 삭제된다"** — 순서는 보장되지 않는다. 먼저 락을 잡은 쪽이 끝날 때까지 다른 쪽이 기다린다.

## 복습 체크

- [ ] 싱글턴 빈의 맵을 동시에 건드리는 스레드 세 종류와 각자의 읽기/쓰기를 말할 수 있는가?
- [ ] HashMap 에서 순회 중 수정 시 무슨 예외가, 어떤 원리(modCount)로 나는가?
- [ ] synchronizedMap 으로 순회를 지키면 생기는 문제를 "락 범위 × 쥐는 시간" 으로 설명할 수 있는가?
- [ ] ConcurrentHashMap 의 get / 빈 칸 put / 노드 있는 칸 쓰기는 각각 어떻게 동기화되는가?
- [ ] ConcurrentHashMap 에서도 기다리는 두 경우는?
- [ ] get→put 로그인 카운터에서 현재 3, 동시 실패 2건이면 최종값은? compute 는 어떻게 막는가?
- [ ] compute 람다에서 하면 안 되는 것 두 가지는?

## 관련

[[race-condition-lost-update]] · [[hashmap-internals]] · [[sse-server-sent-events]] · [[jvm-stack-and-heap]] · [[offline-vs-online-attack]] · [[static-util-vs-spring-bean]]
