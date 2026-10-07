---
type: refined
slug: parallel-stream-pitfalls
tags: [병렬스트림, parallel-stream, parallel, parallelStream, Spliterator, 분할, split, trySplit, ArrayList분할, LinkedList분할, ForkJoinPool, commonPool, 공용풀, 포크조인, fork-join, 블로킹IO, blocking-io, 스레드고갈, thread-starvation, 오버헤드, overhead, 작은데이터, combine, 병렬성능, 코어수, availableProcessors, forEachOrdered, findAny]
topic: parallel() 은 항상 빠르지 않다 — 소스 분할 비용(ArrayList vs LinkedList), 공용 ForkJoinPool, 작은 데이터 오버헤드
summary: 병렬 Stream 은 Spliterator 로 원본을 반씩 쪼개 공용 ForkJoinPool 스레드에 나눠 주고 결과를 합친다. ArrayList·배열은 인덱스 계산으로 O(1) 에 중간을 잘라 바로 일을 나누지만, LinkedList 는 앞에서부터 순차로 걸어가며 떼어내야 해 분할 비용이 병렬 이득을 잡아먹는다. 또 작은 데이터는 분배·합치기 오버헤드가 계산보다 크고, 블로킹 I/O 를 넣으면 앱 전체가 공유하는 commonPool(보통 코어 수-1) 스레드가 묶여 다른 병렬 작업까지 멈춘다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Spliterator.html
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/ForkJoinPool.html
updated: 2026-10-07
---

# 병렬 Stream 의 함정

## 개념 정의

`.parallel()` / `parallelStream()` 은 파이프라인을 여러 스레드로 나눠 돌린다.

1. **분할**: `Spliterator.trySplit()` 으로 원본을 반씩 계속 쪼갠다.
2. **실행**: 조각들을 `ForkJoinPool.commonPool()` 스레드가 나눠 처리한다.
3. **합치기**: 스레드별 결과를 combine 한다 (collect 가 공유 상태 없이 안전한 이유 — [[race-condition-lost-update]]).

`parallel()` 은 이 세 단계 비용을 **추가로 지불**한다. 이득이 비용보다 클 때만 빨라진다.

## 작동 원리 — 손해 보는 세 가지

### 1. 소스 자료구조 — 분할 비용

100만 개를 4등분하려면 중간 지점을 찾아야 한다.

| 소스 | 중간 지점 찾기 | 결과 |
|---|---|---|
| **ArrayList** / 배열 / `IntStream.range` | **O(1)** — 인덱스 계산 ([[arraylist-internals]]) | 즉시 반으로 잘라 4 스레드가 바로 시작 |
| **LinkedList** | **O(n)** — next 를 따라 걸어감 ([[linkedlist-internals]]) | 앞에서부터 순차로 걸으며 덩어리를 떼어내는 동안 다른 스레드는 대기 → 분할 비용이 병렬 이득을 잡아먹음 |

(`HashSet`/`HashMap` 은 버킷 배열 기준으로 나눠 중간 정도, `Stream.iterate` 처럼 순차 생성되는 소스는 분할이 나쁘다.)

### 2. 작은 데이터 — 오버헤드

스레드 분배 + 작업 큐잉 + 결과 합치기 비용이 고정으로 든다. 원소가 적거나 원소당 연산이 가벼우면 **순차보다 느리다.**

### 3. 블로킹 I/O — commonPool 고갈

- 병렬 Stream 은 기본적으로 **JVM 전체가 공유하는** `ForkJoinPool.commonPool()` 을 쓴다. 병렬도는 보통 `코어 수 - 1`.
- 여기서 DB 조회·HTTP 호출처럼 **대기하는 작업**을 하면 몇 개 안 되는 공용 스레드가 전부 놀면서 묶인다 → **앱의 다른 곳에서 돌던 병렬 Stream 까지 멈춘다.**
- I/O 병렬화는 전용 스레드 풀(`ExecutorService`)이나 비동기(`CompletableFuture` + 전용 executor)로 분리한다.

## 트레이드오프 / 한계

| 병렬이 이득 | 병렬이 손해 |
|---|---|
| 큰 데이터 (수만~수백만+) | 작은 데이터 |
| 원소당 **CPU 연산**이 무거움 | 원소당 연산이 가벼움 |
| ArrayList·배열·range 소스 | LinkedList·iterate 소스 |
| 순서 무관 (`findAny`, `unordered`) | 순서 유지 필요 (`forEachOrdered`, `findFirst` 는 비용 증가) |
| 순수 함수 | 블로킹 I/O, 공유 상태 수정 |

- 공유 가변 상태를 람다에서 건드리면 정확성까지 깨진다 (`forEach(list::add)` → 갱신 유실).
- 확신이 없으면 **순차로 두고 측정(JMH)** 한 뒤 결정한다.

> [!WARNING]
> **오답 코너**
> - **"parallel() 을 붙이면 항상 빨라진다"** — 분할·분배·합치기 비용이 추가된다. 소스가 LinkedList 이거나, 데이터가 작거나, 블로킹 I/O 면 오히려 손해다.
> - **"ArrayList/LinkedList 의 중간 지점 찾기는 O(N) / O(N²)"** — **O(1) / O(n)**. 인덱스 계산 vs 한 칸씩 걷기.
> - **"LinkedList 는 next 를 계속 비교해서 느리다"** — 비교가 아니라 **next 를 따라 걸으며 칸 수를 센다.** 손해 보는 단계는 **분할(split)** 이다.
> - **"병렬 Stream 은 자기만의 스레드 풀을 쓴다"** — 기본은 앱 전체 **공용** commonPool 이다.

## 복습 체크

- [ ] 병렬 Stream 의 세 단계(분할·실행·합치기)를 말할 수 있는가?
- [ ] ArrayList 와 LinkedList 소스의 분할 비용 차이를 빅오로 설명할 수 있는가? 손해 보는 단계는?
- [ ] 작은 데이터에서 병렬이 느린 이유는?
- [ ] 블로킹 I/O 를 병렬 Stream 에 넣으면 왜 다른 곳까지 멈추는가?
- [ ] I/O 병렬화는 대신 무엇으로 하는가?

## 관련

[[stream-vs-for-loop]] · [[stream-lazy-evaluation]] · [[race-condition-lost-update]] · [[arraylist-internals]] · [[linkedlist-internals]] · [[big-o-notation]]
