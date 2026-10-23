---
type: refined
slug: integer-overflow
tags: [오버플로, overflow, 정수오버플로, integer-overflow, wraparound, 감겨돎, int범위, int-range, Integer.MAX_VALUE, Integer.MIN_VALUE, 2의31제곱, 32비트, 4바이트, 부호비트, sign-bit, 2의보수, twos-complement, 범위비대칭, long범위, 조용한오류, silent-error, fail-fast, Math.addExact, Math.multiplyExact, Math.toIntExact, ArithmeticException, 금액계산, 음수결제, 면접답변]
topic: "int 의 범위(±21억)는 32비트에서 1비트를 부호로 쓴 결과이고, 넘치면 예외 없이 반대편 끝으로 감겨 돈다 — 틀리면 안 되는 값은 Math.~Exact 로 fail-fast 한다"
summary: "int 는 4바이트(32비트)로 1비트를 부호에, 31비트를 크기에 써서 -2³¹ ~ 2³¹-1(약 ±21억)을 표현하며 0 이 양수 쪽 자리를 차지해 양수가 1 적다. MAX_VALUE + 1 은 올림이 부호 비트까지 넘어가 MIN_VALUE 가 되는데, 자바는 매 연산 검사 비용을 피하려 이 wraparound 를 스펙으로 정하고 예외를 던지지 않는다. 금액처럼 틀리면 안 되는 값은 Math.addExact·multiplyExact·toIntExact 로 넘칠 때 ArithmeticException 을 받는다."
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-4.html#jls-4.2.2
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Math.html#addExact(int,int)
updated: 2026-10-23
---

# 정수 오버플로

## 개념 정의 — int 의 범위는 비트에서 나온다

- `int` = **4바이트 = 32비트**. (8바이트는 `long`, 범위 약 ±922경)
- 1비트는 **부호**, 남은 **31비트**가 크기 → 2³¹ ≈ 21억 4천7백만.

```
Integer.MIN_VALUE = -2,147,483,648   (-2³¹)
Integer.MAX_VALUE =  2,147,483,647   ( 2³¹ - 1)
```

양수 쪽이 1 적은 이유는 **0 이 양수 쪽 자리 하나를 차지**하기 때문이다.

## 작동 원리 — 넘치면 감겨 돈다 (wraparound)

4비트 부호 있는 정수(-8 ~ 7)로 축소해 보면:

```
0111  =  7   ← 최댓값
0001  =  1
────────
1000  = -8   ← 올림이 부호 비트까지 넘어가 최솟값이 됨
```

```java
int max = Integer.MAX_VALUE;
int next = max + 1;
System.out.println(next);   // -2147483648 (Integer.MIN_VALUE)
```

- 컴파일 에러도 런타임 예외도 **없다.** 조용히 반대편 끝으로 넘어간다.
- 왜 검사하지 않나: CPU 덧셈 회로는 원래 넘친 올림을 버린다. 모든 연산마다 검사하면 그만큼 느려지므로, 자바는 **감겨 도는 동작을 언어 스펙으로 정해 두고** 검사하지 않는다.
- 연산 타입이 `int` 인 줄 모르고 `long` 변수에 담는 것만으로는 막을 수 없다 → [[numeric-promotion]].

## 트레이드오프 — 조용한 오류 vs fail-fast

주문 금액 합계를 `int` 로 계산하다 21억을 넘으면 에러 없이 **음수 금액**이 결제 API 로 넘어간다. 틀린 값으로 계속 진행하면 나중에 발견되는 데이터 오염이 된다. 틀리면 안 되는 값은 넘칠 때 멈추게 한다.

```java
Math.addExact(a, b)        // 덧셈이 넘치면 ArithmeticException
Math.multiplyExact(a, b)   // 곱셈이 넘치면 ArithmeticException
Math.toIntExact(longValue) // long → int 축소가 값을 잃으면 ArithmeticException
```

- `toIntExact` 는 **검사하지 않는 약속**인 `(int)` 캐스트([[primitive-type-conversion]])를 **검사하는 약속**으로 바꾼다. 예: JPA `count()` 의 `long` 을 `int` 로 바꿀 때.
- 금액은 애초에 `long`(최소 단위)이나 `BigDecimal` 로 다룬다.

> [!WARNING]
> **오답 코너**
> - **"int 는 8바이트"** — **4바이트(32비트)**. 8바이트면 long 과 범위가 같아야 하는데 long 은 약 922경이다.
> - **"MAX_VALUE + 1 은 예외가 난다"** — 예외 없이 **MIN_VALUE** 가 된다.
> - **"넘치면 예외를 던지는 메서드는 `Math.exactly`"** — `Math.addExact` / `multiplyExact` / `toIntExact`. 접미사가 **Exact** 다.

## 면접 답변

> "int 는 32비트라 1비트를 부호로 쓰고 31비트로 약 ±21억을 표현합니다. 범위를 넘으면 자바는 예외 없이 반대편 끝으로 감겨 돌아서, MAX_VALUE 에 1을 더하면 MIN_VALUE 가 됩니다. 매 연산마다 검사하면 느려지기 때문에 언어 차원에서 검사하지 않는 건데, 금액처럼 틀리면 안 되는 값은 Math.addExact 나 toIntExact 를 써서 넘칠 때 예외가 나게 만듭니다."

## 복습 체크

- [ ] int 의 범위를 비트로 설명하고, 양수 쪽이 1 적은 이유를 말할 수 있는가?
- [ ] 4비트 예제로 `MAX + 1 = MIN` 이 되는 과정을 보일 수 있는가?
- [ ] 자바가 오버플로를 검사하지 않는 이유는?
- [ ] 넘치면 예외가 나게 하는 메서드 세 가지는?
- [ ] 금액 합계에서 조용한 오버플로가 위험한 이유는?

## 관련

[[numeric-promotion]] · [[primitive-type-conversion]] · [[primitive-vs-reference-type]] · [[db-pagination]]
