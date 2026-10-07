---
type: refined
slug: optional-orelse-vs-orelseget
tags: [orElse, orElseGet, Optional, 옵셔널, 즉시평가, eager-evaluation, 지연평가, lazy-evaluation, Supplier, 함수형인터페이스, functional-interface, 람다, lambda, 인자평가순서, argument-evaluation, 부수효과, side-effect, 중복저장, 중복발급, 불필요한호출, 기본값, default-value]
topic: orElse 와 orElseGet 의 차이 — 인자가 언제 실행되는가
summary: Java 는 메서드 호출 전에 인자를 먼저 계산하므로 orElse(expr) 의 expr 은 값이 있어도 항상 실행된다(즉시 평가). orElseGet 은 Supplier 를 받아 비었을 때만 실행한다(지연 평가). 부수효과가 있거나 비싼 기본값은 반드시 orElseGet 에 넣는다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Optional.html#orElseGet(java.util.function.Supplier)
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-15.html#jls-15.12.4.2
updated: 2026-10-07
---

# orElse vs orElseGet

## 개념 정의

```java
Coupon coupon = couponRepository.findById(id).orElse(issueDefaultCoupon());           // A
Coupon coupon = couponRepository.findById(id).orElseGet(() -> issueDefaultCoupon());  // B
```

쿠폰이 **이미 있어도** A 는 `issueDefaultCoupon()` 을 실행하고, B 는 실행하지 않는다.

## 작동 원리

Java 는 메서드를 호출하기 **전에** 인자를 먼저 계산한다. `System.out.println(add(1, 2))` 에서 `add` 가 먼저 실행되고,
`println` 안에서 "필요 없네" 하고 되돌릴 방법은 없다. A 와 B 를 풀어 쓰면:

```java
// A — 즉시 평가 (eager)
Coupon temp = issueDefaultCoupon();                          // ① 값을 "만든다" — 항상 실행
Coupon coupon = couponRepository.findById(id).orElse(temp);  // ②

// B — 지연 평가 (lazy)
Supplier<Coupon> s = () -> issueDefaultCoupon();             // ① "만드는 코드 조각"만 만든다
Coupon coupon = couponRepository.findById(id).orElseGet(s);  // ② 비었을 때만 s.get() 호출
```

B 가 넘기는 것은 `Supplier<Coupon>` — 호출하면 쿠폰을 만들어 주는 코드 조각을 **값처럼** 들고 있는 객체다.

## 트레이드오프 / 한계

- `issueDefaultCoupon()` 이 DB 에 저장한다면, A 는 **조회할 때마다 쓰지도 않을 기본 쿠폰이 1개씩 쌓인다.**
  사용자에겐 "쿠폰함을 열 때마다 쿠폰이 늘어나는" 중복 발급 버그. 외부 API 호출이었다면 매 조회마다 불필요한 네트워크 호출.
- 반대로 **이미 있는 값·상수**라면 `orElse` 가 더 간결하고 람다 객체 생성도 없다.

> 원칙: `orElse` 에는 **이미 만들어진 값·상수**만 (`orElse(0)`, `orElse(Collections.emptyList())`).
> **비용이 크거나 부수효과가 있는 코드**는 `orElseGet`. 같은 원리로 `orElseThrow` 도 `Supplier` 를 받는다.

> [!WARNING]
> **오답 코너**
> - **"값이 있으면 `orElse(issueDefaultCoupon())` 도 호출되지 않는다"** — **항상 호출된다.** 인자는 메서드 호출 전에 계산된다.
> - **"A 로 운영하면 조회마다 2개씩 생긴다"** — 찾은 쿠폰은 원래 있던 것. **새로 생기는 건 기본 쿠폰 1개씩.**
> - **"B 는 함수를 만든다"** — 정확히는 **`Supplier<Coupon>`** 타입의 객체다.

## 복습 체크

- [ ] 값이 있는 Optional 에서 `orElse(f())` 와 `orElseGet(() -> f())` 각각 `f()` 가 실행되는가? 이유는?
- [ ] `orElseGet` 에 넘기는 것의 타입과 역할은?
- [ ] `orElse` 에 부수효과 메서드를 넣었을 때 생기는 실제 버그를 예로 들 수 있는가?
- [ ] `orElse` 를 써도 되는 경우는?

## 관련

[[java-optional]] · [[stream-lazy-evaluation]]
