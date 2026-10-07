---
date: 2026-10-07
type: raw
author: dongju
tags: [Stream, 스트림, 스트림 쓰지 말아야 할 때, when not to use stream, for문 vs 스트림, 부수효과, side effect, checked exception, 체크 예외, 람다 예외, lambda exception, Predicate, Function, 함수형 인터페이스, functional interface, throws, UncheckedIOException, break, findFirst, parallel, 병렬 스트림, parallelStream, Spliterator, 분할, split, ArrayList vs LinkedList, ForkJoinPool, commonPool, 공용 풀, 블로킹 I/O, blocking IO, 면접 답변]
topic: Stream 을 어디에 쓰고 어디에 안 쓰는가 — 부수효과 루프, 람다 안 checked 예외, parallel() 의 함정(분할 비용·commonPool·작은 데이터)
summary: Stream 은 컬렉션을 변환·필터·집계해 결과값을 얻는 도구라, 결과가 아니라 행위(부수효과)가 목적인 반복이나 checked 예외를 던지는 I/O 중심 루프에는 for 문이 낫다. 람다는 함수형 인터페이스 메서드(예 Predicate.test)의 구현인데 그 시그니처에 throws 가 없어 checked 예외를 던질 수 없고 try-catch 로 감싸야 해 선언형 장점이 묻힌다. parallel() 은 ArrayList 처럼 O(1) 분할되는 소스·큰 데이터·CPU 연산에서만 이득이고, LinkedList 소스·작은 데이터·블로킹 I/O(공용 ForkJoinPool 고갈)에선 손해다.
source: session
distilled: true
distilled_at: 2026-10-07
---

## 배운 개념

### 1. 판단 기준

> Stream 은 **"데이터를 변환해서 결과를 얻는"** 도구. 컬렉션 → 새 컬렉션/값.
> "N번 반복해서 **어떤 동작을 실행**" 은 결과가 아니라 **행위(부수효과)** 가 목적 → 선언형·지연·단락이 쓸 데가 없다.

- 고정 10번 반복이 for 문이 나은 진짜 이유는 성능(10번이면 무시할 수준)이 아니라 **목적이 부수효과**이기 때문. `IntStream.range(0, 10).forEach(...)` 도 쓸 수는 있다.

### 2. 람다 안의 checked 예외

```java
for (String path : paths) {
    String content = Files.readString(Path.of(path));   // throws IOException
    if (content.contains("ERROR")) {
        alert(path);
        break;
    }
}
```

- `break` → **`filter` + `findFirst`** (단락 평가)로 표현 가능.
- `IOException` → 람다 안에서 그대로 못 던진다:

```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);    // ← throws 선언이 없다
}
```

람다 본문 = `test()` 의 구현. 구현 메서드는 인터페이스가 선언한 것보다 많은 checked 예외를 던질 수 없다 → "잡거나 선언하거나" 중 **잡기**만 남는다.

```java
paths.stream()
     .filter(path -> {
         try {
             return Files.readString(Path.of(path)).contains("ERROR");
         } catch (IOException e) {
             throw new UncheckedIOException(e);   // unchecked 로 감싸서 탈출
         }
     })
     .findFirst()
     .ifPresent(this::alert);
```

→ for 문보다 길고 읽기 어렵다. **checked 예외를 던지는 I/O 중심 루프는 for 문이 낫다.**

### 3. parallel() 은 항상 빠르지 않다

병렬 Stream 은 `Spliterator` 로 원본을 반씩 계속 쪼개 스레드에 나눈다. 100만 개를 4등분할 때:

| 소스 | 25만 번째 위치 찾기 | 분할 |
|---|---|---|
| **ArrayList** / 배열 | **O(1)** (인덱스 계산) | 즉시 나눔 → 4 스레드 바로 시작 |
| **LinkedList** | **O(n)** (next 따라 걸어감) | 한 스레드가 걸어가는 동안 나머지는 대기 → 분할 비용이 병렬 이득을 잡아먹음 |

추가로 손해 보는 경우:

- **작은 데이터**: 스레드 분배 + 결과 combine 비용 > 계산 비용.
- **블로킹 I/O**: 병렬 Stream 은 앱 전체가 공유하는 `ForkJoinPool.commonPool()` (보통 코어 수 - 1 스레드) 사용 → DB/HTTP 대기를 넣으면 **다른 곳의 병렬 Stream 까지 멈춤.**
- **바깥 공유 상태 수정**: `forEach(list::add)` → 갱신 유실 (→ stream-declarative-shared-state).

### 4. 정리표

| ✅ Stream | ❌ for 문 |
|---|---|
| 컬렉션 변환·필터·집계로 결과값 | 행위(부수효과)가 목적인 반복 |
| 조건 맞는 첫 원소 (`filter` + `findFirst`) | 람다 안 checked 예외가 많은 I/O 로직 |
| 큰 데이터 + CPU 연산 + ArrayList/배열 소스의 병렬 | `parallel()` + 블로킹 I/O / 작은 데이터 / LinkedList |
| | 람다 안에서 바깥 변수·리스트 수정 |

### 5. 면접 답변 뼈대

> "컬렉션을 **변환하거나 걸러서 결과값을 얻는 곳**에는 Stream 을 씁니다. 선언형이라 의도가 바로 읽히고, 중간 결과를 가변 리스트에 쌓지 않아 실수 여지가 적습니다.
> 반대로 **I/O 처럼 checked 예외가 나는 작업**이 중심이거나, 결과가 아니라 **동작 자체가 목적인 반복**에는 for 문을 씁니다. 람다 안에서 예외를 감싸다 보면 오히려 읽기 어려워지기 때문입니다.
> `parallel()` 은 데이터가 크고 CPU 연산 위주일 때만 고려합니다. 공용 ForkJoinPool 을 쓰기 때문에 블로킹 I/O 를 넣으면 다른 작업까지 막힐 수 있어서요."

(TODO: study-board 등 실제 프로젝트 장면 하나 붙이기)

## 오답·헷갈린 점

- **전**: 고정 횟수 루프엔 for 문이 낫다 (이유 미상) → **후**: 성능이 아니라 **결과가 아닌 행위가 목적**이라서.
- **전**: 람다에서 checked 예외를 "던질 수 없다" (이유 없이 결론만) → **후**: `Predicate.test()` 시그니처에 **throws 가 없어서**. (힌트 후 정답)
- **전**: ArrayList / LinkedList 인덱스 위치 찾기 = `O(N)` / `O(N제곱)` → **후**: **O(1) / O(n)**. ⚠️ 같은 날 오전에 배운 `get(i)` 표를 바로 회상 못 함.
- **전**: LinkedList 는 "노드의 next 를 계속 **비교**" 해서 느리다 → **후**: 비교가 아니라 **next 를 따라 걸으며 칸 수를 센다.** ⚠️ 오전 인터뷰와 **같은 표현이 반복**된 오개념.
- **전**: parallel 시 LinkedList 는 "느려진다" (막연) → **후**: **분할(split) 단계**에서 손해 — 한 스레드가 중간 지점까지 걷는 동안 나머지가 논다.

## Q&A

- **Q. Stream 을 어디에 쓰고 어디에 안 쓰나?** A. 위 정리표 + 면접 답변 뼈대.
- **Q. 람다 안에서 IOException 을 못 던지는 이유?** A. 함수형 인터페이스 메서드에 throws 가 없어서. 잡아서 UncheckedIOException 으로 감싸야 한다.
- **Q. break 는 Stream 에서?** A. `filter` + `findFirst` / `anyMatch` (단락).
- **Q. parallel() 붙이면 항상 빠른가?** A. 아니다. LinkedList 소스(분할 O(n)), 작은 데이터(오버헤드), 블로킹 I/O(commonPool 고갈) 에선 손해.

## 다음 학습

- 박싱 비용: `Stream<Integer>` vs `IntStream`, `mapToInt`
- `Collectors.groupingBy` / `toMap` 중복 키 `IllegalStateException`
- 병렬에서 `forEach` vs `forEachOrdered`, `findAny` vs `findFirst`
- 병렬 Stream 을 커스텀 `ForkJoinPool` 에서 돌리는 법과 그 한계
