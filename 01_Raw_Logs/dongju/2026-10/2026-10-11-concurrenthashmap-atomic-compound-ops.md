---
date: 2026-10-11
type: raw
author: dongju
tags: [ConcurrentHashMap, 복합 연산, compound operation, 원자적 연산, atomic operation, compute, computeIfAbsent, computeIfPresent, merge, putIfAbsent, get 후 put, get-then-put, read-modify-write, 읽고 고치고 쓰기, 경쟁 조건, race condition, 갱신 유실, lost update, 로그인 시도 제한, login attempt limiter, rate limit, 무차별 대입, brute force, Recursive update, IllegalStateException, null 금지, 약한 일관성, size 근사치, Redis, 분산 카운터, study-board]
topic: ConcurrentHashMap 이어도 get 후 put 은 원자적이지 않다 — 로그인 실패 카운터의 갱신 유실과 compute/computeIfAbsent 로 읽고-고치고-쓰기를 한 번에
summary: get 과 put 각각은 스레드 안전하지만 그 사이에 틈이 있어, 같은 IP 의 실패 요청 두 개가 동시에 오면 둘 다 3 을 읽고 4 를 써서 최종값이 5 가 아니라 4 가 된다(갱신 유실, read-modify-write). 공격자는 실패를 동시에 몰아 차단 임계값 도달을 늦출 수 있다. compute 는 람다가 도는 동안 그 키의 칸 락을 쥐어 세 단계를 원자적으로 만들고, computeIfAbsent 는 "없으면 만들어 넣기"를 원자적으로 해 동시 첫 구독에서 내부 맵이 두 번 생기지 않게 한다. 람다는 짧게, 같은 맵 재수정 금지, null 금지, 메모리 맵이라 서버 여러 대면 Redis 로.
source: session
distilled: false
---

## 배운 개념

### 1. study-board 의 실제 코드

```java
@Component
public class LoginAttemptLimiter {
    static final int THRESHOLD = 10;
    private final Map<String, LoginAttempts> attemptsByKey = new ConcurrentHashMap<>();

    public void recordFailure(String key) {
        LocalDateTime now = LocalDateTime.now(clock);
        // compute 로 읽고-고치고-쓰기를 한 번에 - 동시에 들어온 실패가 서로를 덮어쓰지 않는다
        LoginAttempts updated = attemptsByKey.compute(key, (ignored, existing) ->
                (existing != null ? existing : LoginAttempts.none())
                        .fail(now, THRESHOLD, WINDOW, BLOCK));
        ...
    }
}
```

### 2. compute 없이 짰다면 — 갱신 유실

```java
LoginAttempts existing = attemptsByKey.get(key);                           // ① 읽기
LoginAttempts updated  = (existing != null ? existing : none()).fail(...);  // ② 고치기
attemptsByKey.put(key, updated);                                            // ③ 쓰기
```

현재 실패 3회, 같은 IP 에서 실패 요청 2개 동시:

```
         스레드 A                                 스레드 B
 t1   ① get → 3
 t2                                             ① get → 3      ← A가 고치기 전 값을 읽음
 t3   ② fail → 4
 t4                                             ② fail → 4
 t5   ③ put → 맵의 값 = 4
 t6                                             ③ put → 맵의 값 = 4   ← A의 결과를 덮어씀
```

- 최종 **4**, 기대값 **5** → 실패 1번이 조용히 사라짐.
- `get`, `put` 각각은 스레드 안전 — 문제는 ①과 ③ **사이의 틈**.
- = **경쟁 조건 → 갱신 유실**, 패턴 **read-modify-write** ([[race-condition-lost-update]]). 10/7 ArrayList 의 `"A"` 소실, 10/9 SimpleDateFormat 날짜 덮어쓰기와 같은 모양.
- 보안 영향: 공격자가 실패 요청을 **동시에 몰아** 보내면 카운트가 덜 올라 임계값(10) 전에 더 많은 비밀번호를 시도할 수 있다.

### 3. compute 가 막는 방법

- `compute(key, 람다)` 는 람다 실행 동안 **그 키가 있는 칸의 락을 쥔다** → B 는 A 의 읽기·고치기·쓰기가 끝날 때까지 대기 → 4 를 읽고 5 를 씀.
- 원자적 복합 연산 도구: `compute`, `computeIfAbsent`, `computeIfPresent`, `merge`, `putIfAbsent`.
- `SseEmitterRegistry.add` 의 `computeIfAbsent(memberId, key -> new ConcurrentHashMap<>())` — "없으면 만들어 넣기" 를 `get` → `put` 으로 나누면 두 탭이 동시에 첫 구독할 때 내부 맵이 두 번 만들어져 한쪽 연결이 사라질 수 있다.

> "스레드 안전한 메서드" 를 조합한다고 스레드 안전해지지 않는다.

### 4. 한계와 주의

- **람다는 짧게**: 도는 동안 그 칸이 잠김. 람다 안에서 I/O 금지, **같은 맵 재수정 금지** (`IllegalStateException: Recursive update` 가능).
- **null 금지**: 키·값 모두 `null` 불가 — `get` 이 `null` 이면 "없음" 이 확실하도록.
- **`size()`·순회는 근사치**: 약한 일관성. `connectedMemberIds()` 도 "그 순간의 대략적인 목록".
- **서버 한 대 기준**: 메모리 맵이라 서버 여러 대면 대수만큼 시도 허용, 재시작하면 초기화. 코드 주석대로 정확한 공유 카운터가 필요하면 **Redis** 로.

### 5. 면접 답변 뼈대 (ConcurrentHashMap 전체)

> "실시간 알림의 SSE 연결 저장소와 로그인 시도 제한 카운터에 ConcurrentHashMap 을 썼습니다. 둘 다 싱글턴 빈의 필드라 여러 요청 스레드가 동시에 추가, 삭제, 순회를 합니다.
> `synchronizedMap` 은 맵 전체에 락 하나라서, 알림을 보내려고 순회하는 동안 맵 전체를 잠근 채 네트워크 전송을 하게 됩니다. 느린 클라이언트 하나 때문에 모든 회원의 구독과 알림이 멈출 수 있죠. ConcurrentHashMap 은 읽기에 락이 없고 쓰기는 칸 단위로만 잠그며, 순회 중 수정도 예외 없이 허용합니다.
> 다만 ConcurrentHashMap 이어도 `get` 후 `put` 은 원자적이지 않아서, 로그인 실패 카운트는 `compute` 로 읽고-고치고-쓰기를 한 번에 처리해 동시 실패가 서로를 덮어쓰지 않게 했습니다."

## 오답·헷갈린 점

- **전**: get→put 동시 실패 2건의 최종값 = "**3개**" / "**2번**" → **후**: 타임라인을 따라가면 **4**(기대값 5). 실패 1번 소실.
- **전**: 이 현상의 이름 = "**기억 안 나**" → **후**: **경쟁 조건 → 갱신 유실**, 패턴 **read-modify-write**. ⚠️ 10/7(ArrayList), 10/9(SimpleDateFormat)에 이어 **이번 주 세 번째 등장**인데 회상 실패 → 복습 우선순위 높음.
- **전**: `compute` 가 막는 원리 → 무응답 → **후**: 람다 실행 동안 그 칸의 락을 쥐어 세 단계를 원자적으로.
- 인터뷰 후반 답변이 짧아짐(피로) — 타임라인 같은 시각 자료에서 이해도가 올라감.

## Q&A

- **Q. ConcurrentHashMap 이면 get 후 put 도 안전한가?** A. 아니다. 각 호출은 안전하지만 사이 틈에서 갱신 유실.
- **Q. 동시 실패 2건, 현재 3 → 최종값?** A. 4 (기대 5).
- **Q. compute 는 어떻게 막나?** A. 람다 동안 칸 락 보유 → 원자적.
- **Q. computeIfAbsent 를 쓴 이유?** A. "없으면 만들어 넣기" 원자화 → 내부 맵 이중 생성·연결 소실 방지.

## 다음 학습

- 분산 환경 카운터: Redis `INCR` + `EXPIRE`, 레이트 리미터 알고리즘(토큰 버킷, 슬라이딩 윈도우)
- `merge` 와 `compute` 의 차이, `LongAdder` 로 카운터
- 이번 주 반복 회상 실패(경쟁 조건·갱신 유실) → `study-review race-condition-lost-update`
