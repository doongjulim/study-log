---
date: 2026-10-07
type: raw
author: dongju
tags: [Stream, 스트림, 지연 평가, lazy evaluation, 중간 연산, intermediate operation, 최종 연산, terminal operation, 파이프라인, pipeline, 수직 처리, vertical processing, 루프 퓨전, loop fusion, 수평 처리, 단락 평가, short-circuit, findFirst, anyMatch, limit, IllegalStateException, 일회용 스트림, stream already operated, IntStream.rangeClosed, peek]
topic: Stream 파이프라인의 실행 모델 — 지연 평가, 수직 처리(루프 퓨전), 단락 평가
summary: filter/map 같은 중간 연산은 Stream 을 반환하며 단계를 등록만 하고, toList/findFirst 같은 최종 연산이 호출돼야 실행이 시작된다(최종 연산이 없으면 아무것도 출력되지 않는다). 실행은 단계별 전체 처리(수평)가 아니라 원소 하나가 파이프라인 끝까지 가는 수직 처리라 원본을 한 번만 돌고 중간 리스트가 없으며, 덕분에 findFirst 등은 답이 나오는 즉시 멈춘다(1~100만 중 첫 7의 배수 → filter 7번).
source: session
distilled: false
---

## 배운 개념

### 1. 지연 평가 — 최종 연산 전엔 아무 일도 안 일어난다

```java
Stream<String> s = Stream.of("a", "b", "c")
        .filter(x -> { System.out.println("filter " + x); return true; })
        .map(x -> { System.out.println("map " + x); return x.toUpperCase(); });

System.out.println("끝");
```

출력: **`끝`** 한 줄뿐.

```java
Stream<T> filter(Predicate<? super T> predicate);                 // Stream 을 반환
<R> Stream<R> map(Function<? super T, ? extends R> mapper);       // Stream 을 반환
```

| 구분 | 예시 | 반환 | 하는 일 |
|---|---|---|---|
| **중간 연산** | `filter`, `map`, `sorted`, `distinct`, `limit` | `Stream` | 파이프라인에 단계 **등록만** |
| **최종 연산** | `toList`, `collect`, `forEach`, `count`, `findFirst` | 결과값 / void | 이때 **실행 시작** |

- 비유: 중간 연산 = **레시피 카드에 단계 적기**. "접시에 담아 줘"(최종 연산) 해야 칼질이 시작된다.
- **일회용**: 최종 연산을 한 번 하면 닫힌다. 같은 `s` 에 다시 최종 연산 → `IllegalStateException: stream has already been operated upon or closed`. 재사용 레시피가 아니라 **일회용 컨베이어 벨트**.

### 2. 수직 처리 (루프 퓨전)

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

- ❌ 수평(filter a,b,c 전부 → map a,b,c 전부) 이라면 filter 결과를 담을 **임시 리스트**가 단계마다 필요 (1억 개면 1억 개짜리).
- ✅ 실제는 **원소 하나가 끝까지** → 다음 원소. 공장 컨베이어: 재료 하나가 검사대 → 가공대 → 포장대를 통과한 뒤 다음 재료.
- 내부적으로는 사실상 for 문 하나:

```java
for (String x : source) {
    if (!filter(x)) continue;   // 1단계
    String y = map(x);          // 2단계
    result.add(y);              // 최종 연산
}
```

→ 단계가 몇 개든 **원본 1회 순회, 중간 컬렉션 없음.**

### 3. 단락 평가 (Short-circuit)

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

- filter 람다는 **7번** 실행. 나머지 999,993 개는 `rangeClosed` 도 지연 생성이라 **만들어지지도 않는다.**
- 단락 연산: `findFirst`, `findAny`, `anyMatch`, `allMatch`, `noneMatch`, `limit`.
- 수평 처리였다면 100만 번 다 돌아야 했다 → **지연 + 수직 + 단락은 한 세트**: 최종 연산이 원소를 하나씩 끌어당기니(pull) 중간 리스트가 필요 없고, 답이 나오면 바로 멈출 수 있다.

## 오답·헷갈린 점

- **전**: 최종 연산 없이도 `filter abc / map ABC / 끝` 이 출력된다 → **후**: 중간 연산은 등록만 — 출력은 **`끝`** 뿐 (반환 타입이 Stream 이라는 힌트 + 레시피 비유로 정정).
- **전**: filter a,b,c 를 전부 처리한 뒤 map a,b,c (수평) 순서가 맞다 → **후**: **수직 처리** — filter a → map a → filter b … (컨베이어 비유로 정정).
- **전**: `findFirst` 면 filter 가 **1번** 실행 → **후**: 조건을 **통과하는 값이 나올 때까지** → 7번. 원리(첫 값만 원하니 멈춘다)는 맞았고 횟수만 틀림.

## Q&A

- **Q. 중간 연산만 있는 Stream 의 출력은?** A. 없음. 최종 연산이 실행을 트리거.
- **Q. 같은 Stream 에 toList 를 두 번 부르면?** A. `IllegalStateException`.
- **Q. filter → map → toList 의 출력 순서는?** A. 원소별로 filter → map 이 번갈아.
- **Q. 1~100만에서 첫 7의 배수 찾기, filter 실행 횟수는?** A. 7번 (단락 평가).

## 다음 학습

- 상태를 가진 중간 연산(`sorted`, `distinct`)은 수직 처리를 어떻게 깨뜨리나 (전체를 모아야 함 → 장벽)
- `peek` 을 디버깅 외에 쓰면 안 되는 이유 (지연 + 최적화로 실행 안 될 수 있음, 예: `count()`)
- `Spliterator` 와 `tryAdvance` 기반 pull 모델
