---
date: 2026-09-26
type: raw
author: dongju
tags: [Optional, 옵셔널, Java 8, null, 널, NullPointerException, NPE, NoSuchElementException, get, orElse, orElseGet, orElseThrow, 즉시 평가, eager evaluation, 지연 평가, lazy evaluation, Supplier, 함수형 인터페이스, 부수효과, side effect, 안티패턴, anti-pattern, 메서드 시그니처, 반환 타입, return type, 파라미터, parameter, 오버로딩, overloading, JPA 엔티티, entity field, Serializable, 직렬화, 빈 컬렉션, empty list, List.of, findById, Spring Data JPA]
topic: Optional 을 왜 쓰는가, 어디까지 강제하는가, 그리고 어디에 두면 안티패턴이 되는가
summary: Optional 은 "없을 수 있다"는 정보를 반환 타입에 새겨 호출자가 무시할 수 없게 할 뿐, 올바른 처리까지 강제하지는 않는다. get() 은 대비 없이 꺼내는 안티패턴이고, orElse 의 인자는 값이 있어도 항상 먼저 실행되므로 부수효과 있는 코드는 orElseGet 에 넣는다. 파라미터·필드·컬렉션 래핑은 상태 수를 늘리거나 매핑이 깨지므로 반환 타입에만 쓴다.
source: session
distilled: true
distilled_at: 2026-09-26
---

## 배운 개념

### 1. 왜 등장했나 (What & Why)

```java
Post findById(Long id);              // A
Optional<Post> findById(Long id);    // B
```

- A: 시그니처만 보면 **"없을 수도 있다"는 정보가 사라진다.** 호출자가 null 체크를 까먹는 건 게을러서가 아니라 **몰라서**다.
  컴파일·테스트는 통과하고 운영에서 NPE 가 나거나, null 이 조용히 흘러가 **엉뚱한 곳에서** 문제가 드러난다.
- B: "없을 수 있음"을 **타입에 새긴다.** `Post` 를 꺼내려면 호출자가 반드시 포장을 뜯는 과정을 거쳐야 한다.

실무 사용 (study-board):

```java
Coupon coupon = couponRepository.findById(id)
        .orElseThrow(CouponNotFoundException::new);   // 없으면 도메인 예외로 바로 연결
```

### 2. Optional 이 강제하는 범위

```java
Optional<Post> opt = postRepository.findById(999L);  // 없는 id
Post post = opt.get();   // 컴파일 됨 → 런타임에 NoSuchElementException
```

- Optional 은 **"없을 수 있다는 사실을 무시할 수 없게"** 강제할 뿐, **처리를 올바르게 했는지**는 개발자 몫이다.
- `get()` 내부는 비었는지 먼저 확인하고 스스로 예외를 던진다:

```java
public T get() {
    if (value == null) {
        throw new NoSuchElementException("No value present");
    }
    return value;
}
```

- NPE = null 참조에 `.` 을 찍었을 때. `NoSuchElementException` = "꺼낼 요소가 없다"(`Iterator.next()` 와 같은 예외).
- Java 10+ 인자 없는 `orElseThrow()` 는 `get()` 과 동작이 같지만 **"없으면 던진다"는 의도가 이름에 드러난다.**

### 3. orElse vs orElseGet — 즉시 평가 vs 지연 평가 (How)

```java
Coupon coupon = couponRepository.findById(id)
        .orElse(issueDefaultCoupon());              // A

Coupon coupon = couponRepository.findById(id)
        .orElseGet(() -> issueDefaultCoupon());     // B
```

풀어 쓰면:

```java
// A
Coupon temp = issueDefaultCoupon();                         // ① 값을 "만든다" — 항상 실행
Coupon coupon = couponRepository.findById(id).orElse(temp); // ②

// B
Supplier<Coupon> s = () -> issueDefaultCoupon();            // ① "만드는 코드 조각"만 만든다
Coupon coupon = couponRepository.findById(id).orElseGet(s); // ② 비었을 때만 s.get() 호출
```

- Java 는 메서드를 호출하기 **전에** 인자를 먼저 계산한다 → A 는 쿠폰이 이미 있어도 `issueDefaultCoupon()` 이 실행된다 (**즉시 평가, eager**).
- B 는 `Supplier<Coupon>` 을 넘기고, Optional 이 비었을 때만 실행된다 (**지연 평가, lazy**).
- `issueDefaultCoupon()` 이 DB 저장을 한다면, A 로 운영 시 **조회할 때마다 쓰지도 않을 기본 쿠폰이 1개씩 쌓인다** → 사용자에겐 "쿠폰함 열 때마다 쿠폰이 늘어나는" 중복 발급 버그. 외부 API 호출이었다면 매 조회마다 불필요한 네트워크 호출.

> 원칙: `orElse` 에는 **이미 만들어진 값·상수**만 (`orElse(0)`, `orElse(Collections.emptyList())`). **비용이 크거나 부수효과가 있는 코드**는 `orElseGet`.

### 4. 위치 안티패턴 (Edge cases)

Optional 은 **"메서드 반환 타입에서, 단일 값이 없을 수 있음을 알리는 용도"** 로 설계됐다.

```java
// ① 엔티티 필드 — ❌
@Entity
class Member {
    private Optional<String> nickname;
}

// ② 메서드 파라미터 — ❌
public void updateProfile(Long id, Optional<String> nickname) { ... }

// ③ 리포지토리 반환 타입 — ✅
Optional<Member> findByEmail(String email);
```

**① 필드:** JPA 는 `Optional` 을 컬럼 타입으로 매핑할 수 없다(매핑 에러). `Optional` 은 `Serializable` 이 아니라 세션 저장·캐시 직렬화에서도 깨진다.

**② 파라미터:** 상태가 3개로 **늘어난다.**

| 호출 | `nickname` 상태 |
|---|---|
| `Optional.of("dongju")` | 값 있음 |
| `Optional.empty()` | 비어 있음 |
| `null` | Optional 자체가 null (컴파일 됨) |

그냥 `String` 이면 값/null 2개. null 문제를 줄이려던 도구가 경우의 수를 늘린다. "선택 입력"은 **오버로딩**으로 표현:

```java
public void updateProfile(Long id) { ... }                   // 닉네임 없이
public void updateProfile(Long id, String nickname) { ... }  // 닉네임 포함
```

**③ 컬렉션 래핑:**

```java
Optional<List<Coupon>> findCouponsByMemberId(Long memberId);   // ❌
List<Coupon> findCouponsByMemberId(Long memberId);             // ✅ 없으면 List.of()
```

- `List` 는 길이 0 으로 "없음"을 이미 표현한다. Optional 로 감싸면 "Optional 이 빔"과 "리스트가 빔" — 같은 뜻의 상태가 2개가 된다.
- Spring Data JPA 도 컬렉션 조회 결과가 없으면 null 이 아니라 빈 리스트를 반환한다.

### 안티패턴 요약표

| 안티패턴 | 왜 문제 | 대안 |
|---|---|---|
| `opt.get()` | 대비 없이 꺼냄 → `NoSuchElementException` | `orElseThrow(도메인 예외)` |
| `orElse(부수효과 호출)` | 값이 있어도 실행 → 중복 저장·불필요한 호출 | `orElseGet(() -> ...)` |
| 파라미터 `Optional<T>` | 상태 3개로 증가, 호출자가 매번 감싸야 함 | 오버로딩 |
| 필드 `Optional<T>` | JPA 매핑 불가, `Serializable` 아님 | nullable 필드 + getter 반환에서만 Optional |
| `Optional<List<T>>` | 빈 리스트와 의미 중복 | 빈 리스트 반환 |

## 오답·헷갈린 점

- **Optional 이 강제하는 범위**
  - 전: "B 는 null 처리를 강제하므로 null 에 더 이상 신경 쓰지 않아도 된다."
  - 후: `get()` 도 컴파일된다. Optional 은 **인지를 강제**할 뿐 **올바른 처리를 강제하지 않는다.**
- **빈 Optional 에서 `get()`**
  - 전: NullPointerException 이 난다.
  - 후: **`NoSuchElementException`**. `get()` 이 비었는지 확인하고 스스로 던진다(null 에 `.` 을 찍는 게 아님).
- **`orElse` 의 인자 실행 시점**
  - 전: 값이 있으면 `orElse(issueDefaultCoupon())` 도 `orElseGet(...)` 도 둘 다 호출되지 않는다.
  - 후: **`orElse` 쪽은 항상 호출된다.** 인자는 메서드 호출 전에 먼저 계산된다(즉시 평가). `orElseGet` 만 지연 평가.
- **`orElse` 버그의 결과**
  - 전: 조회할 때마다 DB 에 2개씩 생긴다.
  - 후: 찾은 쿠폰은 원래 있던 것. **새로 생기는 건 기본 쿠폰 1개씩.**
- **B 가 만드는 것**
  - 전: "함수를 만든다."
  - 후: **`Supplier<Coupon>`** 객체 — 호출하면 쿠폰을 만들어 주는 코드 조각을 값처럼 들고 있는 것.
- **Optional 파라미터의 상태 수**
  - 전: 2가지 상태.
  - 후: **3가지**(값 있음 / empty / Optional 자체가 null). 방어할 "없음"이 2종류이고, 일반 파라미터(2개)보다 오히려 늘어난다.

## Q&A

- **Q. Optional 을 어떻게 쓰고 있나?**
  A. 리포지토리 조회 결과를 받아 `orElseThrow(CouponNotFoundException::new)` 로 도메인 예외에 바로 연결한다.
- **Q. null 을 그냥 반환하던 방식의 진짜 문제는?**
  A. "없을 수 있다"는 정보가 시그니처에 없어서 호출자가 모르고 지나간다. Optional 은 그 정보를 타입에 새긴다.
- **Q. `orElse` 와 `orElseGet` 차이는?**
  A. `orElse` 인자는 값이 있어도 먼저 계산된다(즉시 평가). `orElseGet` 은 `Supplier` 를 받아 비었을 때만 실행한다(지연 평가). 부수효과·고비용 코드는 `orElseGet`.
- **Q. 안티패턴은?**
  A. `get()`, `orElse(부수효과)`, 파라미터·필드·컬렉션에 Optional. 반환 타입에서만 쓴다.
- **30초 면접 답변**
  > "리포지토리 조회 결과는 Optional 로 받아서 orElseThrow 로 바로 도메인 예외로 바꿨습니다. Optional 의 가치는 '없을 수 있다'는 사실을 시그니처에 드러내는 데 있다고 봅니다. 그래서 반환 타입에만 쓰고 필드·파라미터·컬렉션에는 쓰지 않습니다. 파라미터에 쓰면 null 까지 포함해 상태가 3개로 늘어나기 때문입니다. 대비 없이 꺼내는 get() 과, 값이 있어도 인자가 먼저 실행되는 orElse(부수효과) 도 피해야 할 안티패턴이고, 그런 경우는 orElseGet 으로 지연 평가를 합니다."

## 다음 학습

- `Optional.map` / `flatMap` / `filter` 체이닝 — `isPresent()` + `get()` 조합 대신 쓰는 방식과 가독성 트레이드오프
- `ifPresent` / `ifPresentOrElse`(Java 9) — 값이 있을 때만 부수효과를 실행하는 방법
- `OptionalInt` / `OptionalLong` — 원시 타입 박싱 비용 회피
- JPA 엔티티에서 nullable 필드를 getter 반환 시점에만 `Optional` 로 감싸는 패턴의 장단점
