---
type: refined
slug: short-circuit-evaluation
tags: [단락평가, short-circuit-evaluation, 논리연산자, logical-operator, "&&", "||", "&", "|", 비단락, non-short-circuit, 양쪽평가, null가드, null-guard, NullPointerException, NPE, 부수효과, side-effect, 조건식부수효과, 비트연산, bitwise-operator, 비트AND, bitwise-and, 2의거듭제곱, power-of-two, "(n-1)&hash", 짝수판별, even-check, 면접답변]
topic: "&& 는 왼쪽으로 결과가 정해지면 오른쪽을 평가하지 않는 단락 평가이고 & 는 양쪽을 다 평가한다 — 그리고 int 피연산자의 & 는 전혀 다른 비트 AND 다"
summary: "user != null && user.isAdmin() 은 user 가 null 이면 오른쪽을 평가하지 않아 안전하지만, & 는 양쪽을 모두 평가해 isAdmin() 호출에서 NPE 가 난다. boolean 의 & 는 오른쪽 부수 효과가 항상 실행돼야 할 때 쓰지만, 조건식 안의 부수 효과 자체가 나쁜 냄새라 실무에선 드물다. 피연산자가 int 이면 & 는 같은 자리 비트가 둘 다 1일 때만 1을 남기는 비트 AND 로, 나머지 연산이 아니다."
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-15.html#jls-15.23
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-15.html#jls-15.22
updated: 2026-10-23
---

# 단락 평가 — `&&` vs `&`

## 개념 정의

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

null 가드(`x != null && x.method()`)가 성립하는 이유가 단락 평가다.

## 작동 원리 — 부수 효과가 실행되느냐

```java
int count = 0;
boolean r = (count > 5) && (++count > 0);   // count = 0  ← 오른쪽 실행 안 됨
boolean s = (count > 5) &  (++count > 0);   // count = 1  ← 오른쪽도 실행
```

boolean 에 `&` 를 일부러 쓰는 경우는 **오른쪽의 부수 효과가 항상 실행돼야 할 때**다. 하지만 조건식 안에 `++count` 같은 부수 효과를 넣는 것 자체가 읽는 사람을 속이는 코드라, 부수 효과는 조건식 밖으로 빼는 게 보통이다.

## 같은 기호, 다른 연산 — int 의 `&` 는 비트 AND

```
  13   = 1101
&  6   = 0110
────────────
   4   = 0100   ← 같은 자리 비트가 둘 다 1일 때만 1
```

```java
int index = (n - 1) & hash;     // HashMap 버킷 위치
boolean even = (x & 1) == 0;    // 마지막 비트가 0 이면 짝수
```

- `n` 이 **2의 거듭제곱**이면 `n - 1` 은 아래쪽이 전부 1 이라 `(n - 1) & hash` 가 `hash % n` 과 **결과가 같다**(hash ≥ 0). HashMap 이 테이블 크기를 2의 거듭제곱으로 유지하는 이유다 ([[hashmap-internals]]).
- 결과가 같을 뿐 연산은 나머지가 아니다. n 이 2의 거듭제곱이 아니면 다른 값이 나온다.

> [!WARNING]
> **오답 코너**
> - **"int 의 `&` 는 나머지 계산"** — **비트 AND** 다. 2의 거듭제곱으로 나눈 나머지와 결과가 같을 뿐이다.
> - **"`&&` 와 `&` 는 결과만 같으면 아무거나"** — 평가하느냐 마느냐가 다르다. null 가드를 `&` 로 쓰면 NPE.

## 면접 답변

> "&& 는 단락 평가라서 왼쪽이 false 면 오른쪽을 아예 평가하지 않고, & 는 양쪽을 모두 평가합니다. 그래서 user != null && user.isAdmin() 은 null 일 때 안전하지만 & 로 쓰면 NPE 가 납니다. 참고로 피연산자가 정수면 & 는 비트 AND 로, HashMap 은 테이블 크기를 2의 거듭제곱으로 두고 (n-1) & hash 로 나머지 연산을 빠르게 대신합니다."

## 복습 체크

- [ ] `user` 가 null 일 때 `&&` 와 `&` 버전의 결과와 이유는?
- [ ] `||` 의 단락 조건은?
- [ ] boolean 에 `&` 를 일부러 쓰는 경우와, 그게 드문 이유는?
- [ ] `(n - 1) & hash` 가 `hash % n` 과 같아지는 조건은?
- [ ] `(x & 1) == 0` 이 짝수 판별인 이유는?

## 관련

[[hashmap-internals]] · [[operator-evaluation-order]] · [[java-optional]]
