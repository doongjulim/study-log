---
type: refined
slug: reference-vs-value-equality
tags: [==, equals, 동일성, identity, 동등성, equality, 참조비교, reference-equality, 값비교, value-equality, 참조타입, 원시타입, primitive, 래퍼클래스, Objects.equals, NPE, NullPointerException, 널안전, null-safe, 상수풀, 객체비교]
topic: == 는 변수에 담긴 값 자체를 비교할 뿐이며, 참조타입에서 "같은 객체인가"로 보이는 것은 그 값이 참조이기 때문이다
summary: == 의 동작은 한 가지뿐이고 변수에 무엇이 담기느냐에 따라 의미가 달라 보인다. 값이 같은지 물으려면 equals 를 쓰되, null 안전까지 확보하려면 Objects.equals 를 쓴다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Object.html#equals(java.lang.Object)
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Objects.html#equals(java.lang.Object,java.lang.Object)
updated: 2026-09-26
---

# == 와 equals — 동일성과 동등성

## 개념 정의

> **`==` 는 변수라는 상자에 저장된 값 자체를 비교한다. 그 값이 무엇을 의미하는지는 신경 쓰지 않는다.**

| 변수 종류 | 상자에 든 것 | `==` 가 결과적으로 묻는 것 |
|---|---|---|
| 원시타입 | 데이터 그 자체(비트열) | 값이 같은가 |
| 참조타입 | 객체를 가리키는 참조 | **같은 객체인가** (동일성) |

동작은 하나인데 **담긴 것이 달라서 의미가 달라 보일 뿐**이다.
"`==` 는 주소를 비교한다"는 설명은 참조타입이라는 특수 케이스를 전체 정의로 착각한 것이다.

```java
long p = 1000L, q = 1000L;
p == q;   // true — 힙도 객체도 주소도 등장하지 않는다
```

`equals()` 는 **동등성**(값이 같은가)을 묻는 약속이다. 단 `Object.equals()` 의 기본 구현은 `==` 와 동일하므로,
값 비교를 원하면 각 클래스가 재정의해야 한다 ([[equals-hashcode-contract]]).

## 작동 원리

### `String.equals` 는 해시가 아니라 내용을 비교한다

```java
"Aa".hashCode()   // 2112
"BB".hashCode()   // 2112  — 해시는 같다
"Aa".equals("BB") // false — 내용은 다르다
```

해시 비교가 등장하는 자리는 `equals()` 내부가 아니라 `HashMap` 의 버킷 탐색이다.

### `Objects.equals` — `==` 와 `equals` 의 협력

```java
public static boolean equals(Object a, Object b) {
    return (a == b) || (a != null && a.equals(b));
}
```

- `==` 로 먼저 걸러 **둘 다 `null`** 인 경우를 통과시킨다.
- `a != null` 이 앞에 있어 `a.equals()` 는 `a` 가 확실히 `null` 이 아닐 때만 호출된다 → NPE 차단.

`==` 에는 없고 `equals()` 에만 있는 위험이 바로 NPE 다. `==` 는 `null` 이어도 조용히 `false` 를 낸다.

### 유틸리티를 쓰지 않는다면 — 피연산자 순서

`null` 이 아님이 **보장된 쪽을 왼쪽에** 둔다.

```java
if (!loginUser.getId().equals(notice.getWriterId())) { ... }   // 로그인 사용자 ID 는 인증을 통과했으므로 non-null
```

## 트레이드오프 / 한계

- **타입이 다르면 값이 같아도 `false`** 다. `Long.equals()` 의 첫 줄은 `if (obj instanceof Long)` 이므로
  `Integer` 를 넘기면 무조건 `false`. 컴파일 경고도 없다 ([[wrapper-cache-autoboxing]] 의 fail-open 사례).
- 재정의한 `equals` 는 반사성·대칭성·추이성·일관성을 만족해야 한다. `instanceof` 를 쓰면 상속 계층에서
  대칭성이 깨질 여지가 생기는데, 프록시 호환을 위해 그 대가를 받아들이는 경우가 있다 ([[jpa-entity-equality]]).

> [!WARNING]
> **오답 코너**
> - **"`==` 는 힙의 저장 위치를 비교한다"** — 상자에 담긴 **값 자체**를 비교한다.
>   원시타입은 힙이 없어도 `==` 가 동작한다(`long p == q` → `true`).
> - **"`equals()` 는 값의 해시를 비교한다"** — 내용을 비교한다. `"Aa"` 와 `"BB"` 는 해시가 같아도 `false`.
> - **"NPE 를 피하려면 `Optional` 을 쓴다"** — `java.util.Objects.equals(a, b)` 다.

## 복습 체크

- [ ] `==` 를 힙·주소라는 단어 없이 한 문장으로 정의할 수 있는가?
- [ ] `long p = 1000L; long q = 1000L;` 에서 `p == q` 가 `true` 인 이유를 그 정의로 설명할 수 있는가?
- [ ] `"Aa".equals("BB")` 가 `false` 인데 해시는 같은 이유는?
- [ ] `Objects.equals` 의 세 줄이 각각 어떤 경우를 막는가?
- [ ] `equals` 를 직접 쓸 때 어느 피연산자를 왼쪽에 두어야 하는가?

## 관련

[[equals-hashcode-contract]] · [[wrapper-cache-autoboxing]] · [[jpa-entity-equality]] · [[string-immutability]] · [[jvm-stack-and-heap]] · [[java-optional]]
