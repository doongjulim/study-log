---
type: refined
slug: stream-lazy-evaluation
tags: [Stream, 스트림, 지연평가, lazy-evaluation, 중간연산, intermediate-operation, 최종연산, terminal-operation, 파이프라인, pipeline, 수직처리, vertical-processing, 루프퓨전, loop-fusion, 수평처리, 단락평가, short-circuit, findFirst, findAny, anyMatch, allMatch, noneMatch, limit, 일회용스트림, IllegalStateException, stream-has-already-been-operated, pull, 상태있는연산, stateful-operation, sorted, distinct, peek]
topic: Stream 파이프라인은 최종 연산이 원소를 하나씩 끌어당기는 구조 — 지연 평가, 수직 처리(루프 퓨전), 단락 평가가 한 세트다
summary: filter/map 같은 중간 연산은 Stream 을 반환하며 단계를 등록만 하고, 최종 연산이 호출돼야 실행이 시작된다(최종 연산이 없으면 람다가 한 번도 안 돈다). 실행은 단계별 전체 처리(수평)가 아니라 원소 하나가 파이프라인 끝까지 가는 수직 처리라 원본을 한 번만 돌고 중간 컬렉션이 없다. 덕분에 findFirst 같은 단락 연산은 답이 나오는 즉시 멈춰, 1~100만 중 첫 7의 배수를 찾을 때 filter 는 7번만 실행된다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/package-summary.html#StreamOps
updated: 2026-10-07
---

# Stream 지연 평가 · 수직 처리 · 단락 평가

## 개념 정의

```java
Stream<String> s = Stream.of("a", "b", "c")
        .filter(x -> { System.out.println("filter " + x); return true; })
        .map(x -> { System.out.println("map " + x); return x.toUpperCase(); });

System.out.println("끝");
```

출력: **`끝`** 한 줄뿐. 람다는 한 번도 실행되지 않았다.

```java
Stream<T> filter(Predicate<? super T> predicate);            // Stream 을 반환
<R> Stream<R> map(Function<? super T, ? extends R> mapper);  // Stream 을 반환
```

| 구분 | 예시 | 반환 | 하는 일 |
|---|---|---|---|
| **중간 연산** | `filter`, `map`, `sorted`, `distinct`, `limit` | `Stream` | 파이프라인에 단계를 **등록만** |
| **최종 연산** | `toList`, `collect`, `forEach`, `count`, `findFirst`, `anyMatch` | 결과값 / void | 이때 **실행 시작** |

- 비유: 중간 연산은 **레시피 카드에 단계를 적는 것**이다. "접시에 담아 줘"(최종 연산)라고 해야 칼질이 시작된다.
- **일회용**: 최종 연산을 한 번 하면 닫힌다. 다시 쓰면 `IllegalStateException: stream has already been operated upon or closed`. 재사용 레시피가 아니라 **일회용 컨베이어 벨트**다.
- 같은 "지연 평가" 발상: `Optional.orElseGet(Supplier)` ([[optional-orelse-vs-orelseget]]).

## 작동 원리

### 1. 수직 처리 (루프 퓨전)

```java
List<String> result = Stream.of("a", "b", "c")
        .filter(x -> { System.out.println("filter " + x); return true; })
        .map(x -> { System.out.println("map " + x); return x.toUpperCase(); })
        .toList();
```

```
filter a
map a
filter b
map b
filter c
map c
```

- ❌ 수평(filter a,b,c 전부 → map a,b,c 전부)이라면 단계마다 결과를 담을 **임시 리스트**가 필요하다. 원소가 1억 개면 1억 개짜리 리스트가 단계마다 생긴다.
- ✅ 실제로는 **원소 하나가 끝까지** 간 뒤 다음 원소가 온다. 공장 컨베이어에서 재료 하나가 검사대 → 가공대 → 포장대를 통과한 다음 다음 재료가 오는 것과 같다.

사실상 이런 for 문 **하나**로 합쳐진다:

```java
for (String x : source) {
    if (!filter(x)) continue;   // 1단계
    String y = map(x);          // 2단계
    result.add(y);              // 최종 연산
}
```

→ 단계가 몇 개든 **원본 1회 순회, 중간 컬렉션 없음.**

### 2. 단락 평가 (Short-circuit)

```java
OptionalInt first = IntStream.rangeClosed(1, 1_000_000)
        .filter(n -> n % 7 == 0)
        .findFirst();
```

```
n=1 → filter false
 ...
n=6 → filter false
n=7 → filter true  → findFirst 가 받음 → 즉시 종료 🛑
```

- filter 람다는 **7번** 실행된다. 나머지 999,993 개는 `rangeClosed` 도 지연 생성이라 **만들어지지도 않는다.**
- 단락 연산: `findFirst`, `findAny`, `anyMatch`, `allMatch`, `noneMatch`, `limit`.
- for 문의 `break` 에 해당한다 ([[stream-vs-for-loop]]).

### 셋은 한 세트

**최종 연산이 원소를 하나씩 끌어당긴다(pull)** → 그래서 중간 리스트가 필요 없고(수직) → 답이 나오면 그 자리에서 멈출 수 있다(단락). 최종 연산이 없으면 끌어당길 주체가 없으니 아무것도 안 돈다(지연).

## 트레이드오프 / 한계

- **상태를 가진 중간 연산**(`sorted`, `distinct`)은 수직 흐름을 끊는다. `sorted` 는 모든 원소를 받아 봐야 첫 원소를 낼 수 있어 내부에 전부 모은다(장벽). `sorted` 뒤의 `findFirst` 는 단락 이득이 줄어든다.
- **부수효과를 넣으면 안 되는 이유**: 실행 여부·횟수·순서를 라이브러리가 결정한다. 예: Java 9+ 의 `count()` 는 원본 크기를 알 수 있으면 파이프라인을 **아예 실행하지 않아** `peek`/`map` 안의 출력이 사라질 수 있다. `peek` 은 디버깅용으로만.
- 지연 덕분에 무한 스트림(`Stream.iterate`, `generate`)도 `limit` 과 함께 쓸 수 있다.

> [!WARNING]
> **오답 코너**
> - **"최종 연산이 없어도 filter/map 람다는 실행된다"** — 중간 연산은 Stream 을 돌려주며 **등록만** 한다. 최종 연산이 없으면 출력은 0줄이다.
> - **"filter 를 전부 끝낸 뒤 map 을 전부 한다"(수평)** — **원소 하나씩 끝까지** 간다(수직). 그래서 중간 리스트가 없다.
> - **"findFirst 면 filter 가 1번만 실행된다"** — 조건을 **통과하는 값이 나올 때까지** 실행된다. 첫 7의 배수면 7번이다. "첫 값만 원하니 멈춘다"는 원리는 맞다.

## 복습 체크

- [ ] 중간 연산과 최종 연산을 반환 타입으로 구분할 수 있는가?
- [ ] 최종 연산 없는 파이프라인의 출력을 예측할 수 있는가?
- [ ] 같은 Stream 에 최종 연산을 두 번 부르면 무슨 예외가 나는가?
- [ ] filter → map → toList 의 출력 순서와, 수평이 아닌 이유(중간 리스트)를 설명할 수 있는가?
- [ ] 1~100만 중 첫 7의 배수 찾기에서 filter 실행 횟수와 그 이유는?
- [ ] 지연·수직·단락이 왜 한 세트인지 "pull" 로 설명할 수 있는가?
- [ ] `sorted` 가 수직 흐름을 끊는 이유는?

## 관련

[[stream-vs-for-loop]] · [[optional-orelse-vs-orelseget]] · [[parallel-stream-pitfalls]] · [[big-o-notation]]
