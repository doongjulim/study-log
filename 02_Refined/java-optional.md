---
type: refined
slug: java-optional
tags: [Optional, 옵셔널, Java8, null, 널, null안전, NullPointerException, NPE, NoSuchElementException, get, orElseThrow, isPresent, 메서드시그니처, 반환타입, return-type, 안티패턴, anti-pattern, Optional파라미터, Optional필드, 오버로딩, overloading, JPA엔티티, Serializable, 직렬화, 빈컬렉션, empty-list, List.of, findById, Spring-Data-JPA, 도메인예외]
topic: Optional 이 해결하는 문제, 실제로 강제하는 범위, 그리고 어디에 두면 안티패턴이 되는가
summary: Optional 은 "없을 수 있다"는 정보를 반환 타입에 새겨 호출자가 무시할 수 없게 할 뿐, 올바른 처리까지 강제하지는 않는다(get 은 컴파일되고 NoSuchElementException 을 던진다). 반환 타입 전용으로 설계됐으므로 파라미터(상태 3개로 증가)·필드(JPA 매핑 불가·비직렬화)·컬렉션 래핑(빈 리스트와 의미 중복)은 안티패턴이다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Optional.html
updated: 2026-09-26
---

# Optional

## 개념 정의

```java
Post findById(Long id);              // A
Optional<Post> findById(Long id);    // B
```

- **A 의 진짜 문제는 NPE 가 아니라 정보 손실이다.** 시그니처만 봐서는 "없을 수도 있다"는 걸 알 수 없다.
  호출자가 null 체크를 까먹는 건 게을러서가 아니라 **몰라서**다. 컴파일·테스트는 통과하고,
  운영에서 NPE 가 나거나 null 이 조용히 흘러가 **엉뚱한 곳에서** 문제가 드러난다.
- **B 는 그 정보를 타입에 새긴다.** `Post` 를 꺼내려면 호출자가 반드시 포장을 뜯어야 한다.

실무 기본 패턴:

```java
Coupon coupon = couponRepository.findById(id)
        .orElseThrow(CouponNotFoundException::new);   // 없으면 도메인 예외로 바로 연결
```

## 작동 원리

### Optional 이 강제하는 범위

Optional 은 **"없을 수 있다는 사실을 무시할 수 없게"** 강제할 뿐, **올바르게 처리했는지**는 개발자 몫이다.

```java
Post post = postRepository.findById(999L).get();   // 컴파일 됨 → 런타임 NoSuchElementException
```

```java
// Optional.get() 내부
public T get() {
    if (value == null) {
        throw new NoSuchElementException("No value present");
    }
    return value;
}
```

- NPE = null 참조에 `.` 을 찍었을 때. `NoSuchElementException` = "꺼낼 요소가 없다" (`Iterator.next()` 와 같은 예외).
- `get()` 으로 대비 없이 꺼내면 null 체크 안 한 옛날 코드와 같다. 터지는 예외 이름만 바뀐다.
- Java 10+ 인자 없는 `orElseThrow()` 는 `get()` 과 동작이 같지만 **"없으면 던진다"는 의도가 이름에 드러난다.**

### 꺼내는 방법

| 메서드 | 비어 있을 때 | 비고 |
|---|---|---|
| `orElseThrow(XxxException::new)` | 지정 예외 | ✅ 기본 패턴 |
| `orElse(값)` | 값 반환 | 인자는 **항상 먼저 계산** → [[optional-orelse-vs-orelseget]] |
| `orElseGet(() -> ...)` | Supplier 실행 | 비었을 때만 실행 |
| `get()` | `NoSuchElementException` | ❌ 안티패턴 |

## 트레이드오프 / 한계 — 위치 안티패턴

Optional 은 **"메서드 반환 타입에서, 단일 값이 없을 수 있음을 알리는 용도"** 로 설계됐다.

**① 필드** — ❌

```java
@Entity
class Member { private Optional<String> nickname; }
```

JPA 는 `Optional` 을 컬럼 타입으로 매핑할 수 없다. `Optional` 은 `Serializable` 도 아니라 세션·캐시 직렬화에서 깨진다.
대안: nullable 필드 + getter 반환 시점에만 `Optional.ofNullable(nickname)`.

**② 파라미터** — ❌ 상태가 오히려 늘어난다.

```java
public void updateProfile(Long id, Optional<String> nickname) { ... }
```

| 호출 | `nickname` 상태 |
|---|---|
| `Optional.of("dongju")` | 값 있음 |
| `Optional.empty()` | 비어 있음 |
| `null` | Optional 자체가 null (컴파일 됨) |

그냥 `String` 이면 값/null 2개. null 을 줄이려던 도구가 경우의 수를 3개로 늘린다. 선택 입력은 **오버로딩**으로:

```java
public void updateProfile(Long id) { ... }
public void updateProfile(Long id, String nickname) { ... }
```

**③ 컬렉션 래핑** — ❌

```java
Optional<List<Coupon>> findCouponsByMemberId(Long memberId);   // ❌
List<Coupon> findCouponsByMemberId(Long memberId);             // ✅ 없으면 List.of()
```

`List` 는 길이 0 으로 "없음"을 이미 표현한다. 감싸면 "Optional 이 빔"과 "리스트가 빔"이 같은 뜻의 두 상태가 된다.
Spring Data JPA 도 컬렉션 조회 결과가 없으면 null 이 아니라 빈 리스트를 반환한다.

| 안티패턴 | 왜 문제 | 대안 |
|---|---|---|
| `opt.get()` | 대비 없이 꺼냄 | `orElseThrow(도메인 예외)` |
| `orElse(부수효과 호출)` | 값이 있어도 실행 | `orElseGet` |
| 파라미터 `Optional<T>` | 상태 3개로 증가 | 오버로딩 |
| 필드 `Optional<T>` | JPA 매핑 불가, 비직렬화 | nullable 필드 + getter 에서만 Optional |
| `Optional<List<T>>` | 빈 리스트와 의미 중복 | 빈 리스트 반환 |

> [!WARNING]
> **오답 코너**
> - **"Optional 을 쓰면 null 에 더 이상 신경 쓰지 않아도 된다"** — `get()` 도 컴파일된다. Optional 은 **인지를 강제**할 뿐 **올바른 처리를 강제하지 않는다.**
> - **"빈 Optional 에서 `get()` 하면 NPE"** — **`NoSuchElementException`**. `get()` 이 비었는지 확인하고 스스로 던진다.
> - **"Optional 파라미터는 상태가 2개"** — **3개**(값 / empty / Optional 자체가 null). 일반 파라미터보다 늘어난다.
> - **"NPE 를 피하려면 Optional"** — 비교 상황이라면 답은 `Objects.equals` 다 ([[reference-vs-value-equality]]).

## 복습 체크

- [ ] `Post findById` 와 `Optional<Post> findById` 의 차이를 "시그니처가 전달하는 정보" 관점에서 설명할 수 있는가?
- [ ] Optional 이 강제하는 것과 강제하지 않는 것은?
- [ ] 빈 Optional 에서 `get()` 을 호출하면 나는 예외와, 그게 NPE 가 아닌 이유는?
- [ ] Java 10 인자 없는 `orElseThrow()` 가 추가된 이유는?
- [ ] Optional 파라미터가 안티패턴인 이유를 상태 개수로 설명하고 대안을 말할 수 있는가?
- [ ] 엔티티 필드에 Optional 을 두면 생기는 문제 두 가지는?
- [ ] `Optional<List<T>>` 대신 무엇을 반환해야 하는가?

## 관련

[[optional-orelse-vs-orelseget]] · [[reference-vs-value-equality]] · [[checked-vs-unchecked-exception]] · [[derived-query-method]]
