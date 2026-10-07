---
type: refined
slug: race-condition-lost-update
tags: [경쟁조건, race-condition, 레이스컨디션, 갱신유실, lost-update, 데이터유실, 공유가변상태, shared-mutable-state, 스레드안전, thread-safety, 동시성, concurrency, 원자성, atomicity, read-modify-write, check-then-act, size++, ArrayList동시add, ArrayIndexOutOfBoundsException, 병렬, parallel, collect, combiner, synchronizedList, CopyOnWriteArrayList, AtomicInteger, CAS]
topic: 공유 가변 상태에 여러 스레드가 "읽고-고치고-쓰기"를 하면 갱신이 조용히 유실된다 — ArrayList 동시 add 로 보는 경쟁 조건
summary: ArrayList.add 는 elementData[size] 에 넣고 size++ 하는 두 단계라 원자적이지 않다. 두 스레드가 같은 size 를 읽으면 같은 칸을 덮어써 한 값이 사라지고 size 도 실제 add 횟수보다 작아진다(갱신 유실). 에러 없이 데이터만 사라져 찾기 어렵다. 해법은 공유 상태를 없애거나(스레드별 컨테이너 후 합치기 — Stream collect), 원자적으로 만들거나(락, CAS, 동시성 컬렉션)다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/ArrayList.html
  - https://docs.oracle.com/javase/tutorial/essential/concurrency/interfere.html
updated: 2026-10-07
---

# 경쟁 조건과 갱신 유실

## 개념 정의

- **경쟁 조건(Race Condition)**: 결과가 스레드 실행 **타이밍(누가 먼저냐)** 에 따라 달라지는 상황.
- **갱신 유실(Lost Update)**: 그 결과 한 스레드의 쓰기가 다른 스레드의 쓰기에 **덮여 사라지는** 것.

뿌리는 **힙의 공유 객체**다. 지역변수는 스레드마다 스택에 따로 있어 안전하지만, 힙의 객체는 모든 스레드가 함께 본다 ([[jvm-stack-and-heap]]).

## 작동 원리 — ArrayList 동시 add

`ArrayList.add()` 는 사실상 두 단계다 ([[arraylist-internals]]):

```java
elementData[size] = e;   // ① 현재 size 위치에 쓰기
size++;                  // ② size 증가 (이것도 읽기 → +1 → 쓰기 3단계)
```

`size = 5` 에서 두 스레드:

```
         스레드 1 (제목 "A")              스레드 2 (제목 "B")
 t1   size 읽음 → 5
 t2                                    size 읽음 → 5
 t3   elementData[5] = "A"
 t4                                    elementData[5] = "B"
 t5   size = 5 + 1 → 6
 t6                                    size = 5 + 1 → 6
```

| 확인 | 결과 | 기대 |
|---|---|---|
| `elementData[5]` | `"B"` | A, B 둘 다 |
| `"A"` | **사라짐** | 존재 |
| `size` | 6 | 7 |

- **예외도 로그도 없다.** 데이터만 조용히 사라진다 → 가장 찾기 어려운 부류의 버그.
- 배열이 꽉 차서 확장(grow)되는 순간에 겹치면 `ArrayIndexOutOfBoundsException` 이 나기도 한다.
- 패턴 이름: **read-modify-write** (읽고 → 고치고 → 쓰기 사이에 끼어듦). 비슷한 것으로 **check-then-act** ([[string-concatenation-cost]] 의 StringBuffer 예).

## 해법 — 두 갈래

| 갈래 | 방법 | 예 |
|---|---|---|
| **공유를 없앤다** | 스레드마다 자기 컨테이너에 모으고 마지막에 합침 | Stream `collect` / `toList` 의 병렬 처리 (accumulator + combiner) |
| **원자적으로 만든다** | 읽기-쓰기를 한 덩어리로 묶음 | `synchronized`, `Collections.synchronizedList`, `CopyOnWriteArrayList`, `AtomicInteger`(CAS), `ConcurrentHashMap` |

- Stream 이 안전한 이유는 Stream 이라서가 아니라 **공유를 안 만드는 방식**이라서다. `forEach(list::add)` 로 바깥 리스트를 건드리면 다시 깨진다 ([[stream-vs-for-loop]]).

## 트레이드오프 / 한계

- 락은 정확성을 사고 **처리량**을 내준다. 개별 메서드만 동기화하면 메서드 사이(check-then-act)는 여전히 깨진다.
- 동시성 컬렉션도 **복합 연산**까지 원자적이진 않다 (`if (!map.containsKey(k)) map.put(k, v)` → `putIfAbsent`/`computeIfAbsent` 를 써야 함).
- 웹 요청 단위에서도 같은 구조가 나타난다 — 동시 토큰 갱신 경합 ([[token-refresh-concurrency]]).

> [!WARNING]
> **오답 코너**
> - **"여러 스레드가 같은 리스트에 add 하면 순서가 꼬인다"** — 순서만이 아니다. 같은 size 를 읽어 **같은 칸을 덮어쓴다** → 값 소실 + size 불일치(갱신 유실).
> - **"`size++` 는 한 줄이니 원자적이다"** — 읽기 → +1 → 쓰기 3단계다.
> - **"Stream 을 쓰면 스레드 안전하다"** — collect 가 안전한 것이지, 람다 안에서 공유 상태를 바꾸면 똑같다.

## 복습 체크

- [ ] `ArrayList.add` 가 원자적이지 않은 이유를 두 단계로 말할 수 있는가?
- [ ] size=5 에서 두 스레드 타임라인을 그리고 결과 세 가지(칸 내용, 사라진 값, size)를 말할 수 있는가?
- [ ] 경쟁 조건과 갱신 유실의 관계는?
- [ ] 해법 두 갈래(공유 제거 vs 원자화)와 각각의 예는?
- [ ] 동시성 컬렉션을 써도 깨지는 복합 연산 예를 들 수 있는가?

## 관련

[[jvm-stack-and-heap]] · [[arraylist-internals]] · [[stream-vs-for-loop]] · [[token-refresh-concurrency]] · [[string-concatenation-cost]]
