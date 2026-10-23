---
type: refined
slug: numeric-promotion
tags: [산술승격, numeric-promotion, binary-numeric-promotion, 이항숫자승격, 연산타입, operand-type, 피연산자타입, 연산전승격, int로승격, byte덧셈, byte+byte, short, char, long곱셈, 50000곱하기50000, 리터럴타입, long리터럴, 캐스트위치, cast-precedence, 상수표현식, constant-expression, 복합대입연산자, compound-assignment, +=, 숨은캐스트, 정수나눗셈, integer-division, 버림, truncation, 0쪽으로버림, 비율계산, 성공률, 퍼센트, double캐스트, 계산전형변환, 면접답변]
topic: "연산 타입은 결과를 담을 변수가 아니라 연산 순간 피연산자의 타입이 정한다 — long 에 담아도 int 곱셈은 넘치고, double 에 담아도 int 나눗셈은 버린다"
summary: "byte·short·char 는 연산 전에 int 로 승격되고, 두 피연산자 중 넓은 쪽 타입으로 연산된다. 그래서 long total = 50_000 * 50_000 은 int 곱셈에서 이미 넘쳐 -1794967296 이고, double rate = success / total * 100 은 int 나눗셈 3/4 = 0 이라 0.0 이다. 고치려면 연산 전에 피연산자 하나를 넓은 타입으로 만든다((long) a * b, (double) success / total). (long)(a * b) 처럼 연산 뒤에 캐스트하면 소용없다. byte c = a + b 는 결과가 int 라 컴파일 에러지만 리터럴끼리는 상수 표현식이라 허용되고, += 는 숨은 캐스트로 조용히 감겨 돈다."
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-5.html#jls-5.6
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-15.html#jls-15.17.2
updated: 2026-10-23
---

# 산술 승격 — 연산 타입은 피연산자가 정한다

## 개념 정의

- `int` 보다 작은 타입(`byte`, `short`, `char`)은 산술 연산 **전에** `int` 로 승격된다.
- 두 피연산자의 타입이 다르면 **넓은 쪽**(int < long < float < double)으로 맞춰 연산한다.

> **결과를 담을 변수의 타입이 아니라, 연산 순간 피연산자의 타입이 연산 타입을 정한다.**

## 작동 원리

### long 에 담아도 넘친다

```java
long total = 50_000 * 50_000;
System.out.println(total);   // -1794967296
```

```
① 50_000 * 50_000     int × int → int. 25억은 int 에 안 들어감 → 여기서 이미 넘침
② total = (그 int 값)  int → long 확대. 손실 없이 깨진 값을 그대로 넓힘
```

| 코드 | 결과 | 이유 |
|---|---|---|
| `long a = (long) (50_000 * 50_000);` | -1794967296 | 캐스트가 곱셈 **뒤**. 괄호 안 int 곱셈이 먼저 넘침 |
| `long b = (long) 50_000 * 50_000;` | 2500000000 | 캐스트가 `*` 보다 먼저 결합 → 첫 피연산자가 long → long 곱셈 |
| `long c = 50_000L * 50_000;` | 2500000000 | long 리터럴 → long 곱셈 |

### double 에 담아도 버린다 — 정수 나눗셈

정수 나눗셈은 반올림하지 않고 **0 쪽으로 버린다.**

```
 3 / 4  =  0.75  →  0
 7 / 2  =  3.5   →  3
-7 / 2  = -3.5   → -3     (내림이면 -4)
```

```java
int success = 3, total = 4;
double rate = success / total * 100;
// ① 3 / 4 → int 나눗셈 → 0   ② 0 * 100 → 0   ③ double 대입 → 0.0
```

에러·경고 없이 성공률 75% 가 **0.0** 이 된다. 통계 화면의 "성공률 0%"로 늦게 발견되는 버그다.

```java
double rate = (double) success / total * 100;   // 75.0 — 캐스트 하나면 충분
double rate = success * 100.0 / total;          // 75.0 — double 리터럴로 시작
double rate = (double) (success / total) * 100; // 0.0 ❌ — 괄호 안 int 나눗셈이 먼저
```

### byte 끼리 더했는데 컴파일 에러

```java
byte a = 10, b = 20;
byte c = a + b;      // ❌ byte + byte → int + int → 결과 int → byte 로 축소 불가
byte c1 = 10 + 20;   // ✅ 상수 표현식 — 컴파일러가 값(30)을 알고 범위 안이라 허용
```

컴파일러가 막는 근거는 **값이 아니라 타입**이다. `a`, `b` 는 변수라 실행 중 120 + 120 이 올 수도 있다.

## 트레이드오프 — `+=` 의 숨은 캐스트

```java
byte c2 = 120;
c2 += 10;            // ✅ 컴파일됨 — c2 = (byte)(c2 + 10)
System.out.println(c2);   // -126  ← 경고 없이 조용히 감겨 돎
```

복합 대입 연산자는 편하지만 **명시적 캐스트를 숨긴 것**이라 오버플로([[integer-overflow]])를 눈에서 가린다.

> [!WARNING]
> **오답 코너**
> - **"`long total = 50_000 * 50_000` 은 total 에 담길 때 잘린다"** — 잘림은 **int 곱셈 안에서** 일어난다. int → long 확대는 값을 잃지 않는다. 이렇게 이해하면 `(long)(a * b)` 같은 틀린 수정을 하게 된다.
> - **"long 을 명시했으니 된다"** — `(long)(50_000 * 50_000)` 도 명시했지만 실패한다. 기준은 **곱하기 전에** long 이 됐느냐다.
> - **"`byte + byte` 에러는 연산 후 변환 순서 때문"** — **산술 승격**: 연산 전에 int 로 올라가고, 결과 int → byte 축소를 컴파일러가 막는다.
> - **"`3 / 4` 는 반올림해서 1"** — **0**. 정수 나눗셈은 0 쪽으로 버린다.
> - **"`success / total * 100` 은 0.75"** — `* 100` 을 빠뜨린 값이라 어떤 경우에도 나올 수 없다. 실제는 **0.0**.

## 면접 답변

> "자바는 연산 타입을 결과를 담을 변수가 아니라 피연산자로 정합니다. 그래서 int 끼리 곱한 값을 long 변수에 담으면 곱셈 단계에서 이미 넘치고, int 끼리 나눈 값을 double 에 담으면 소수점이 이미 버려집니다. 그래서 연산 전에 피연산자 하나를 long 이나 double 로 바꿔야 합니다."

## 복습 체크

- [ ] `long total = 50_000 * 50_000` 이 음수인 이유를 두 단계로 말할 수 있는가?
- [ ] A·B·C 세 가지 수정 중 맞는 것과 그 이유는?
- [ ] `double rate = success / total * 100` 의 값과 고치는 방법은?
- [ ] `byte c = a + b` 는 에러인데 `byte c = 10 + 20` 과 `c += a` 는 되는 이유는?
- [ ] 정수 나눗셈 `-7 / 2` 의 결과는?

## 관련

[[integer-overflow]] · [[primitive-type-conversion]] · [[operator-evaluation-order]] · [[primitive-vs-reference-type]]
