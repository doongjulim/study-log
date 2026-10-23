---
date: 2026-10-23
type: raw
author: dongju
tags: [단락평가, short-circuit-evaluation, 논리연산자, logical-operator, "&&", "||", "&", "|", 비단락, non-short-circuit, 양쪽평가, null가드, null-guard, NullPointerException, NPE, 부수효과, side-effect, 비트연산, bitwise-operator, 비트AND, bitwise-and, 2의거듭제곱, power-of-two, 나머지최적화, modulo, HashMap인덱스, "(n-1)&hash", 짝수판별, even-check, 마지막비트, 면접답변]
topic: "&& 는 왼쪽으로 결과가 정해지면 오른쪽을 평가하지 않는 단락 평가이고 & 는 양쪽을 다 평가한다 — 그리고 int 피연산자의 & 는 전혀 다른 비트 AND 다"
summary: "user != null && user.isAdmin() 은 user 가 null 이면 오른쪽을 평가하지 않아 안전하지만, & 는 양쪽을 모두 평가해 isAdmin() 호출에서 NPE 가 난다. boolean 의 & 는 오른쪽 부수 효과가 항상 실행돼야 할 때 쓰지만 조건식 안의 부수 효과 자체가 나쁜 냄새다. 피연산자가 int 이면 & 는 같은 자리 비트가 둘 다 1일 때만 1을 남기는 비트 AND 이고, n 이 2의 거듭제곱일 때 (n-1) & hash 가 hash % n 과 같은 결과를 내므로 HashMap 이 테이블 크기를 2의 거듭제곱으로 유지한다."
source: session
distilled: true
distilled_at: 2026-10-23
---

## 배운 개념

### 1. `&&` vs `&` — 단락 평가

```java
User user = null;

if (user != null && user.isAdmin()) { ... }   // (A) 정상 — 오른쪽을 평가하지 않음
if (user != null &  user.isAdmin()) { ... }   // (B) NPE — 오른쪽도 평가해 isAdmin() 호출
```

| | `&&` / `\|\|` | `&` / `\|` (boolean) |
|---|---|---|
| 평가 방식 | **단락 평가(short-circuit)** — 결과가 정해지면 오른쪽 생략 | **양쪽 모두** 평가 |
| `&&` | 왼쪽 `false` → 결과 `false` 확정 → 오른쪽 생략 | |
| `\|\|` | 왼쪽 `true` → 결과 `true` 확정 → 오른쪽 생략 | |

- null 가드(`x != null && x.method()`)가 성립하는 이유가 단락 평가.

### 2. boolean 에 `&` 를 쓰는 경우 — 오른쪽 부수 효과가 항상 실행돼야 할 때

```java
int count = 0;
boolean r = (count > 5) && (++count > 0);   // count = 0  ← 오른쪽 실행 안 됨
boolean s = (count > 5) &  (++count > 0);   // count = 1  ← 오른쪽도 실행
```

- 하지만 조건식 안에 `++count` 같은 **부수 효과**를 넣는 것 자체가 읽는 사람을 속이는 코드. 부수 효과는 조건식 밖으로 뺀다 → 실무에서 boolean `&` 는 드물다.

### 3. int 피연산자의 `&` — 비트 AND (다른 연산)

```
  13   = 1101
&  6   = 0110
────────────
   4   = 0100   ← 같은 자리 비트가 둘 다 1일 때만 1
```

```java
int index = (n - 1) & hash;     // HashMap 버킷 위치 계산
boolean even = (x & 1) == 0;    // 짝수 판별
```

- `n` 이 **2의 거듭제곱**이면 `n - 1` 은 아래쪽이 전부 1 (`16 - 1 = 15 = 0000 1111`) → `hash & 15` 는 아래 4비트만 남겨 0~15 → `hash % 16` 과 같은 결과(hash ≥ 0 일 때).
- 그래서 HashMap 은 테이블 크기를 2의 거듭제곱으로 유지하고 **2배**로 늘린다 — 느린 `%` 대신 빠른 비트 AND ([[hashmap-internals]]).
- `x & 1` 은 마지막 비트만 남긴다. 이진수의 마지막 비트가 0이면 짝수.

### 면접 답변

> "&& 는 단락 평가라서 왼쪽이 false 면 오른쪽을 아예 평가하지 않고, & 는 양쪽을 모두 평가합니다. 그래서 user != null && user.isAdmin() 은 null 일 때 안전하지만 & 로 쓰면 NPE 가 납니다. 참고로 피연산자가 정수면 & 는 비트 AND 로, HashMap 은 테이블 크기를 2의 거듭제곱으로 두고 (n-1) & hash 로 나머지 연산을 빠르게 대신합니다."

## 오답·헷갈린 점

- 👍 (A) 정상 / (B) NPE — 결과와 이유 정답. 용어 **단락 평가(short-circuit evaluation)** 만 보충받음.
- 👍 `&` 가 필요한 경우 = "앞의 조건과 상관없이 새로운 조건이 항상 검사되어야 할 때" — 정답. (힌트 코드의 count 값 0 / 1 은 답하지 않아 보충받음)
- **전**: int 피연산자의 `&` = "**나머지 계산**할 때" → **후**: **비트 AND**. 2의 거듭제곱으로 나눈 나머지와 **결과가 같을 뿐**(HashMap 의 최적화). 연산 자체를 나머지로 이해하면 n 이 2의 거듭제곱이 아닐 때 틀린 결과를 믿게 된다.

## Q&A

**Q. `&&` 와 `&` 의 차이는?**
A. `&&` 는 단락 평가(결과가 정해지면 오른쪽 생략), `&` 는 양쪽 모두 평가. null 가드는 `&&` 여야 한다.

**Q. boolean 에 `&` 를 일부러 쓰는 경우는?**
A. 오른쪽의 부수 효과가 항상 실행돼야 할 때. 단 그런 부수 효과는 조건식 밖으로 빼는 게 낫다.

**Q. `(n - 1) & hash` 는 무엇을 하나?**
A. 비트 AND. n 이 2의 거듭제곱이면 hash % n 과 같아서 HashMap 이 버킷 인덱스를 빠르게 구하는 데 쓴다.

## 다음 학습

- 비트 연산자 전체(`|`, `^`, `~`, `<<`, `>>`, `>>>`)와 HashMap 의 `hash ^ (hash >>> 16)` 섞기
- 비트 플래그(권한 마스크)와 `EnumSet`
