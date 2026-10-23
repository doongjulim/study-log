---
type: refined
slug: primitive-type-conversion
tags: [형변환, type-conversion, 캐스팅, casting, 묵시적형변환, implicit-conversion, 자동형변환, 명시적형변환, explicit-conversion, 확대변환, widening, widening-primitive-conversion, 축소변환, narrowing, narrowing-primitive-conversion, 표현범위, 비트절단, truncation, 캐스트는약속, 검사하지않는약속, toIntExact, 정밀도, precision, 유효숫자, mantissa, significand, 지수, exponent, IEEE754, float, double, 16777217, 2의24제곱, 2의53제곱, 0.1+0.2, BigDecimal, 금액계산, 면접답변]
topic: "기본형 형변환은 바이트 크기가 아니라 표현 범위로 자동/명시가 갈리고, 캐스트는 아무도 검사하지 않는 약속이며, 확대 변환도 정밀도는 잃을 수 있다"
summary: "표현 범위가 좁은 타입에서 넓은 타입으로는 자동(확대 변환), 반대는 (타입) 캐스트가 필요(축소 변환)하며 축소는 위쪽 비트를 잘라 (int) 3_000_000_000L 이 -1294967296 이 된다. 캐스트는 값을 고쳐 주는 장치가 아니라 컴파일러도 런타임도 검사하지 않는 약속이다. 확대 변환도 int/long → float/double 은 지수에 비트를 쓰는 대가로 정밀도를 잃어 16_777_217 이 16_777_216 이 되므로, 금액은 double 대신 long 이나 BigDecimal 로 다룬다."
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-5.html#jls-5.1.2
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-5.html#jls-5.1.3
updated: 2026-10-23
---

# 기본형 형변환 — 확대와 축소

## 개념 정의 — 기준은 표현 범위

```
byte → short → int → long → float → double
        (char → int)
좁은 → 넓은: 자동 (확대 변환, widening)
넓은 → 좁은: (타입) 필요 (축소 변환, narrowing)
```

```java
int  i = 100;
long l = i;            // ✅ 확대 — 자동
long big = 100L;
int  j = big;          // ❌ 컴파일 에러 — 축소인데 캐스트 없음
int  k = (int) big;    // ✅ 명시적 캐스트
```

- 기준은 **바이트 크기가 아니라 표현 범위**다. `long`(8바이트) → `float`(4바이트)도 자동이다(float 범위 약 ±3.4×10³⁸).
- 기본형↔래퍼 변환은 형변환이 아니라 **박싱/언박싱**이다 ([[unboxing-npe-wrapper-null]]).

## 작동 원리 — 캐스트는 "검사하지 않는 약속"

```java
long big = 3_000_000_000L;   // 30억 (int 범위 밖)
int  k   = (int) big;
System.out.println(k);       // -1294967296  ← 위쪽 32비트를 잘라내고 남은 비트를 int 로 읽음
```

- `(int)` 는 값을 정상으로 바꿔 주는 장치가 아니다. "**잘려도 내가 책임진다**"는 서명이다.
- 컴파일러도 런타임도 검사하지 않는다 → 약속이 깨지면 에러 없이 틀린 값이 들어간다.
- 검사하는 버전은 `Math.toIntExact` ([[integer-overflow]]).

## 트레이드오프 — 확대 변환도 정밀도는 잃는다

```java
int   big = 16_777_217;
float f   = big;              // 캐스트 없이 자동 (확대 변환)
System.out.println((int) f);  // 16777216  ← 1이 사라짐
```

```
float (32비트) = 부호 1비트 + 지수 8비트 + 유효숫자 23비트(+ 숨은 1비트 = 24비트 정밀도)
```

- 지수에 비트를 써서 **범위**를 얻고, 유효숫자는 24비트뿐이라 **정밀도**를 잃는다. 2²⁴ = 16,777,216 까지만 모든 정수를 정확히 표현한다.
- 확대 변환은 **범위** 손실은 없지만 `int`/`long` → `float`/`double` 은 **정밀도** 손실이 가능하다. 그런데도 자동으로 허용된다(`long` → `double` 은 2⁵³ 초과 시).
- `0.1 + 0.2 == 0.30000000000000004` 도 같은 원리다(0.1 이 이진수로 딱 떨어지지 않음). **금액은 `double` 금지** — `long`(원 단위)이나 `BigDecimal`.

> [!WARNING]
> **오답 코너**
> - **"하위 계층 → 상위 계층은 자동"** — 자바에서 "계층"은 상속을 연상시킨다. 기준은 **표현 범위**(좁은 → 넓은)다.
> - **"직접 캐스팅하면 정상적으로 진행된다"** — 캐스트는 값을 고쳐 주지 않는다. `(int) 3_000_000_000L == -1294967296`.
> - **"캐스트는 값이 범위를 넘지 않는다는 약속"** — 방향은 맞지만, 그 약속을 **아무도 검사하지 않는다**는 점까지 말해야 한다.
> - **"확대 변환은 항상 손실이 없다"** — 범위는 그렇지만, 정수 → 실수는 **정밀도**를 잃을 수 있다.

## 면접 답변

> "표현 범위가 좁은 타입에서 넓은 타입으로는 자동으로 변환되고, 반대는 명시적 캐스트가 필요합니다. 캐스트는 위쪽 비트를 그냥 잘라내기 때문에 값이 바뀌어도 에러가 나지 않습니다. 그리고 int 나 long 을 float, double 로 바꾸는 건 자동이지만 정밀도를 잃을 수 있어서 금액 계산에는 BigDecimal 을 씁니다."

## 복습 체크

- [ ] 확대·축소 변환의 순서를 쓰고, 기준이 바이트 크기가 아닌 이유를 `long → float` 로 설명할 수 있는가?
- [ ] `(int) 3_000_000_000L` 의 결과와 이유는?
- [ ] 캐스트를 "검사하지 않는 약속"이라고 부르는 이유는?
- [ ] `16_777_217` 을 float 로 바꿨다 되돌리면 왜 값이 바뀌는가?
- [ ] 금액을 double 로 계산하면 안 되는 이유는?

## 관련

[[numeric-promotion]] · [[integer-overflow]] · [[primitive-vs-reference-type]] · [[unboxing-npe-wrapper-null]]
