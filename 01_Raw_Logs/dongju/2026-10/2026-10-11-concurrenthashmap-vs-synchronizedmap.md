---
date: 2026-10-11
type: raw
author: dongju
tags: [ConcurrentHashMap, 컨커런트해시맵, synchronizedMap, Collections.synchronizedMap, Hashtable, synchronized, 락, lock, 락 범위, lock granularity, 칸 단위 락, bin lock, CAS, compare-and-swap, volatile, 락 없는 읽기, lock-free read, ConcurrentModificationException, fail-fast, modCount, 약한 일관성, weakly consistent, 순회 중 수정, 싱글턴 빈 공유 상태, shared mutable state, SseEmitter, SSE, SseEmitterRegistry, LoginAttemptLimiter, 리사이즈, resize, help transfer, 스레드 안전, thread safety, 면접 답변, study-board]
topic: ConcurrentHashMap 을 쓴 이유와 HashMap 에 synchronized 를 거는 것(synchronizedMap)과의 차이 — 락 범위(맵 전체 vs 칸), 락 없는 읽기, 순회 중 수정
summary: study-board 의 SseEmitterRegistry 와 LoginAttemptLimiter 는 싱글턴 빈의 필드라 구독(추가)·타임아웃(삭제)·알림 발송(순회+삭제)이 서로 다른 스레드에서 동시에 일어난다. HashMap 이면 동시 쓰기에서 엔트리가 사라지고, 순회 중 수정은 fail-fast 이터레이터가 ConcurrentModificationException 을 던진다(send 는 단일 스레드에서도 순회 중 스스로 remove). synchronizedMap 은 맵 전체에 락 하나라 순회를 지키려면 맵 전체를 잠근 채 네트워크 전송을 해야 해 모든 회원이 멈춘다. ConcurrentHashMap(Java 8+)은 get 에 락이 없고, 빈 칸은 CAS, 노드가 있는 칸은 첫 노드에만 synchronized 를 걸며, 순회는 약한 일관성이라 예외 없이 돈다.
source: session
distilled: true
distilled_at: 2026-10-11
---

## 배운 개념

### 1. study-board 의 사용처

| 클래스 (`@Component`) | 필드 | 용도 |
|---|---|---|
| `SseEmitterRegistry` | `Map<Long, Map<Long, SseEmitter>> emittersByMember` | 회원별 SSE 실시간 알림 연결(탭) 관리 |
| `LoginAttemptLimiter` | `Map<String, LoginAttempts> attemptsByKey` | IP별 로그인 실패 횟수 → 무차별 대입 차단 |

```java
@Component
public class SseEmitterRegistry {
    private final Map<Long, Map<Long, SseEmitter>> emittersByMember = new ConcurrentHashMap<>();

    public SseEmitter add(Long memberId) {
        ...
        Map<Long, SseEmitter> emitters =
                emittersByMember.computeIfAbsent(memberId, key -> new ConcurrentHashMap<>());
        emitters.put(emitterId, emitter);
        emitter.onCompletion(() -> emitters.remove(emitterId));
        emitter.onTimeout(() -> emitters.remove(emitterId));
        emitter.onError(e -> emitters.remove(emitterId));
        ...
    }

    public void send(Long memberId, String eventName, String eventId, Object data) {
        Map<Long, SseEmitter> emitters = emittersByMember.get(memberId);
        if (emitters == null) return;
        emitters.forEach((emitterId, emitter) -> {
            try {
                emitter.send(SseEmitter.event().id(eventId).name(eventName).data(data));
            } catch (IOException | IllegalStateException e) {
                emitters.remove(emitterId); // 끊어진 클라이언트 정리
            }
        });
    }
}
```

### 2. 누가 동시에 건드리나

싱글턴 빈 → 필드는 힙에 하나 → 모든 스레드가 공유.

| 상황 | 메서드 | 맵에 하는 일 |
|---|---|---|
| ① 새 탭이 알림 구독 (요청 스레드 A) | `add` → `computeIfAbsent`, `put` | 쓰기 (추가) |
| ② 예전 탭 30분 타임아웃 (컨테이너 스레드 B) | `onTimeout` → `remove` | 쓰기 (삭제) |
| ③ 댓글 알림 발송 (요청 스레드 C) | `send` → `forEach`, 실패 시 `remove` | 읽기(순회) + 가끔 쓰기 |

### 3. HashMap 이면

- 동시 쓰기 → 경쟁 조건으로 엔트리가 조용히 사라짐 ([[race-condition-lost-update]]).
- 순회 중 수정 → `HashMap` 은 `modCount`(구조 변경 횟수)를 기억했다가 다음 원소로 갈 때 비교 → 다르면 **`ConcurrentModificationException`** (fail-fast 이터레이터). fail-fast 는 최선 노력일 뿐 보장은 아님.
- **`send()` 는 순회 도중 스스로 `remove`** → HashMap 이면 **다른 스레드 없이도** CME. ConcurrentHashMap 의 이터레이터는 **약한 일관성(weakly consistent)** 이라 순회 중 추가·삭제가 있어도 예외 없이 계속 돈다.

### 4. synchronizedMap 의 한계

```java
Map<Long, SseEmitter> emitters = Collections.synchronizedMap(new HashMap<>());
```

- `get`/`put`/`remove` **메서드 하나하나**를 맵 전체 락 하나로 감쌈 → 각 호출은 안전.
- **순회는 못 지킴**: `forEach` 는 "다음 꺼내기" 반복이고, 락은 호출 사이마다 풀린다 → 틈에 `remove` 가 끼면 CME (10/9 StringBuffer 의 check-then-act 와 같은 구조).
- 공식 문서 권장대로 `synchronized (map) { 순회 }` 로 감싸면 → 그 안에서 `emitter.send()` = **네트워크 전송** → 느린 클라이언트 하나에 2초면 그 2초 동안 **모든 회원**의 구독·타임아웃 정리·알림이 대기.
- **락 비용 = 범위(맵 전체) × 쥐는 시간(네트워크 I/O)**.

### 5. ConcurrentHashMap (Java 8+) 구조

10/7 HashMap 구조(`Node[] table` + 칸마다 체인) 위에서:

| 동작 | `synchronizedMap` | `ConcurrentHashMap` |
|---|---|---|
| `get` | 맵 전체 락 | **락 없음** (volatile 읽기) |
| 빈 칸에 `put` | 맵 전체 락 | **CAS** 로 노드 끼워 넣기 |
| 노드가 있는 칸에 `put`/`remove` | 맵 전체 락 | **그 칸의 첫 노드에만** `synchronized` |
| 순회 | 직접 `synchronized(map)`, 아니면 CME | 락 없이 약한 일관성 |

- CAS: "지금 값이 내가 본 값 그대로면 새 값으로, 아니면 실패" 를 CPU 명령 하나로 원자적으로. 실패하면 재시도.
- 비유: synchronizedMap = **맵 전체가 줄 하나**, ConcurrentHashMap = **칸마다 각자의 줄**, 읽기는 줄을 서지 않음.

동시 요청 예 — A: 회원 7 등록(`computeIfAbsent(7, …)`), B: 회원 9 타임아웃 `remove`:

| | 결과 |
|---|---|
| synchronizedMap | 먼저 락을 잡은 쪽이 끝날 때까지 다른 쪽 대기 (순서 보장 없음) |
| ConcurrentHashMap (다른 칸) | 동시에 진행 |
| ConcurrentHashMap 에서도 기다리는 경우 | **같은 칸에 쓰기**끼리(같은 키 / 해시 충돌), 또는 **리사이즈 중**(쓰려는 스레드가 옮기기를 도움 — help transfer) |

## 오답·헷갈린 점

- **전**: `SseEmitterRegistry` 는 "**로그인 유지 정보**를 담는 코드" → **후**: SSE **실시간 알림 연결**(회원별 탭) 관리. 같이 본 `LoginAttemptLimiter` 와 섞임. (7월 SSE 학습 내용과 연결 못 함)
- **전**: ② 타임아웃 = **읽기**, ③ 알림 발송 = **쓰기** → **후**: 반대. 타임아웃은 연결 정리라 **삭제(쓰기)**, 발송은 **순회(읽기)** + 끊긴 연결 삭제.
- **전**: 순회 중 수정 시 "값이 달라져 있으면 **에러를 뱉는다**" → **후**: 방향 맞음, 이름까지 — **`ConcurrentModificationException`** (fail-fast, modCount).
- **전**: synchronizedMap 에서는 "**등록이 완전히 끝난 후 삭제**" → **후**: 순서 보장 없음. 먼저 락을 잡은 쪽이 끝날 때까지 다른 쪽 대기.
- **전**: ConcurrentHashMap 에서도 "**같은 맵에서 읽기·쓰기가 동시에**" 일어나면 기다린다 → **후**: `get` 은 **락이 없다.** 기다리는 건 같은 칸 쓰기끼리, 리사이즈 중뿐.
- **정답 맞힘**: 메서드 단위 락으론 순회를 못 막는다 / 전체 락 + 네트워크 전송이면 다른 작업이 멈춘다 / 다른 칸이면 동시 진행.

## Q&A

- **Q. ConcurrentHashMap 을 쓴 이유는?** A. 싱글턴 빈의 공유 맵에 추가·삭제·순회가 서로 다른 스레드에서 동시에 일어나서.
- **Q. HashMap 에 synchronized 를 거는 것과 뭐가 다른가?** A. 락 범위가 맵 전체 vs 칸, 읽기 락 유무, 순회 중 수정 허용 여부. synchronizedMap 으로 순회를 지키면 맵 전체를 잠근 채 네트워크 전송을 하게 된다.
- **Q. ConcurrentHashMap 에서 기다리는 경우는?** A. 같은 칸에 쓰기끼리, 리사이즈 중.

## 다음 학습

- `ConcurrentHashMap` 리사이즈(transfer, ForwardingNode)와 여러 스레드가 함께 옮기는 방식
- Java 7 이전 `Segment` 기반 구조와 Java 8 이후 비교
- CAS 와 `AtomicLong`(`SseEmitterRegistry.sequence`), ABA 문제
