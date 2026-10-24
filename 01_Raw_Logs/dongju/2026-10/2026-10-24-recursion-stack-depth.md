---
date: 2026-10-24
type: raw
author: dongju
tags: [재귀, recursion, 재귀호출, recursive-call, 종료조건, base-case, 스택프레임, stack-frame, 호출스택, call-stack, StackOverflowError, Error, Exception, ExceptionHandler, 재귀깊이, recursion-depth, Xss, 스택크기, 꼬리호출최적화, tail-call-optimization, 반복문변환, iteration, for문, factorial, 팩토리얼, 오버플로, overflow, BigInteger, 면접답변]
topic: 재귀는 호출마다 스택 프레임이 쌓이므로 종료 조건이 없으면 StackOverflowError 가 나고, 종료 조건이 있어도 깊이가 크면 넘친다 — 깊이가 입력에 비례하면 반복문으로 바꾼다
summary: 재귀 호출마다 매개변수·지역 변수·돌아갈 위치를 담은 스택 프레임이 하나씩 쌓이고 스택은 유한하다. 종료 조건(base case)이 없으면 끝없이 쌓여 StackOverflowError 가 나는데, 이는 Exception 이 아니라 Error 라 @ExceptionHandler(Exception.class) 에 걸리지 않는다. 종료 조건이 있어도 factorial(100_000) 처럼 도달 전에 프레임을 10만 개 쌓아야 하면 스택(-Xss, 보통 수백 KB~1MB)이 넘치고, 자바는 꼬리 호출 최적화가 없어 재귀 깊이가 곧 프레임 개수다. 깊이가 입력에 비례하면 반복문으로 바꾸고, 큰 수 계산이면 오버플로도 따로 챙긴다(13! 부터 int 초과, BigInteger).
source: session
distilled: true
distilled_at: 2026-10-24
---

## 배운 개념

### 1. 종료 조건이 없으면 — StackOverflowError

```java
static int factorial(int n) {
    return n * factorial(n - 1);      // 종료 조건 없음
}
factorial(5);   // 5 → 4 → … → 0 → -1 → … 영원히 내려감
```

- 호출마다 **스택 프레임**(매개변수 `n`, 지역 변수, 돌아갈 위치)이 하나씩 쌓인다. 스택은 유한하다.
- 꽉 차면 **`StackOverflowError`** — `Exception` 이 아니라 **`Error`** → `@ExceptionHandler(Exception.class)` 에 걸리지 않고 500 까지 올라간다 ([[jvm-stack-and-heap]]).

```java
static int factorial(int n) {
    if (n <= 1) return 1;          // 종료 조건(base case)
    return n * factorial(n - 1);
}
```

### 2. 종료 조건이 있어도 깊이가 크면 넘친다

- `factorial(100_000)` 은 종료 조건에 **닿기 전에** 프레임을 10만 개 쌓아야 한다.
- 스레드 스택은 보통 수백 KB~1MB(`-Xss` 로 설정) → 그 전에 넘친다.
- 자바는 마지막 호출의 프레임을 재사용하는 **꼬리 호출 최적화가 없다** → 재귀 깊이 = 프레임 개수.

```java
static long factorial(int n) {
    long result = 1;
    for (int i = 2; i <= n; i++) result *= i;   // 프레임 1개로 끝
    return result;
}
```

- 반복문으로 바꿔도 `factorial(100_000)` 은 `int`·`long` 범위를 한참 넘는다(13! 부터 int 초과) → 조용히 감겨 돈 틀린 값 → 큰 수는 `BigInteger` ([[integer-overflow]]).

### 면접 답변

> "재귀는 호출할 때마다 스택 프레임이 쌓이기 때문에 종료 조건이 없으면 StackOverflowError 가 납니다. 이건 Exception 이 아니라 Error 라 일반 예외 처리로 잡히지 않습니다. 종료 조건이 있어도 입력이 커서 깊이가 깊어지면 같은 문제가 생기고 자바는 꼬리 호출 최적화도 없어서, 깊이가 입력 크기에 비례하는 경우에는 반복문으로 바꿉니다."

## 오답·헷갈린 점

- 👍 "호출할 때마다 메서드가 스택에 쌓인다, 스택은 무한하지 않다" — 원인 정답. 정확한 용어는 **스택 프레임**이 쌓인다. 빠진 것(종료 조건)과 결과의 이름(`StackOverflowError`, `Error`)은 보충.
- 👍 큰 깊이 대안 = "for 문으로 변경" — 정답. 왜 종료 조건이 있어도 위험한지(도달 전 깊이, 꼬리 호출 최적화 없음)와 오버플로는 보충.

## Q&A

**Q. 재귀에 종료 조건이 없으면?**
A. 호출마다 스택 프레임이 쌓여 StackOverflowError. Exception 이 아니라 Error.

**Q. 종료 조건이 있어도 위험한 경우는?**
A. 깊이가 커서 종료 조건에 닿기 전에 스택이 넘칠 때. 자바는 꼬리 호출 최적화가 없다 → 반복문으로.

## 다음 학습

- 재귀가 자연스러운 경우(트리 순회, 분할 정복)와 명시적 스택(`ArrayDeque`)으로 바꾸기
- `-Xss` 와 스레드 수의 관계(스레드마다 스택 하나: [[jvm-stack-and-heap]])
