---
type: refined
slug: wrapper-cache-autoboxing
tags: [오토박싱, autoboxing, 언박싱, unboxing, 래퍼클래스, wrapper, Integer, Long, valueOf, IntegerCache, LongCache, 정수캐시, integer-cache, -128~127, 객체재사용, 식별자비교, 소유자확인, 권한검사, fail-open, fail-closed, 타입불일치, contains, 엔티티식별자, GeneratedValue]
topic: -128~127 만 캐시된 래퍼 객체를 재사용하기 때문에, == 로 ID 를 비교하면 값이 커지는 순간 조용히 깨진다
summary: Long a = 100L 은 Long.valueOf 로 컴파일되어 캐시된 같은 객체를 돌려주지만 128 이상은 매번 새 객체다. == 가 127까지 우연히 통과하다 128부터 실패하므로 식별자 비교는 반드시 equals 를 써야 한다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-5.html#jls-5.1.7
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Long.html#valueOf(long)
updated: 2026-09-10
---

# 오토박싱과 래퍼 캐시

## 개념 정의

`Long a = 100L;` 은 컴파일러가 **`Long.valueOf(100L)`** 호출로 바꾼다. `new Long(100L)` 이 아니다.

```java
public static Long valueOf(long l) {
    if (l >= -128 && l <= 127) return LongCache.cache[(int)l + 128]; // 미리 만들어둔 객체 재사용
    return new Long(l);                                             // 범위 밖은 새 객체
}
```

JVM 이 클래스 로딩 시점에 -128~127 의 객체 256개를 미리 만들어 배열에 넣어둔다.
**-128~127 은 부호 있는 8비트(`byte`)의 표현 범위**이며, 원시타입 `long` 의 범위(약 ±922경)와는 무관하다.

목적은 메모리·GC 최적화다. 0, 1, 200 같은 작은 정수는 압도적으로 자주 쓰이므로
매번 `new` 하면 힙에 쓰레기가 쌓인다. 자주 쓰이는 소수의 값만 미리 만들어 공유한다.

## 작동 원리

```
Long a = 100L, b = 100L;
a 객체 정체성 : 148912029
b 객체 정체성 : 148912029   ← 완전히 동일. 두 개가 아니라 하나.
a == b : true

Long c = 1000L, d = 1000L;
c == d      : false
c.equals(d) : true

Long x = new Long(100L), y = new Long(100L);   // 캐시 강제 우회
x == y      : false
x.equals(y) : true
```

**가장 중요한 해석**: 128 미만에서 `==` 가 `true` 인 것은 "`==` 가 값 비교를 해준 것"이 아니라
**"우연히 같은 객체였던 것"**이다. `==` 는 두 경우 모두 똑같이 동작했고,
달라진 것은 `valueOf` 가 객체를 재사용했는지 여부뿐이다 ([[reference-vs-value-equality]]).

`Long` 변수에는 범위와 무관하게 **항상 참조가 담긴다.** 범위에 따라 달라지는 것은
"무엇이 저장되는가"가 아니라 "그 참조가 어떤 객체를 가리키는가"다.

## 트레이드오프 / 한계

### 왜 엔티티 `@Id` 는 래퍼 타입인가

원시타입 `long` 은 `null` 을 담지 못해 "아직 채번 전"을 `0` 으로밖에 표현하지 못한다.
Spring Data JPA 의 `AbstractEntityInformation.isNew()` 는 ID 가 `null` 이거나
**원시 숫자이면서 값이 `0`** 일 때 새 엔티티로 판단하므로, `0` 은 "미할당"인지 "진짜 0번 행"인지 구분이 불가능해진다.
작성자가 없는 익명 글 같은 상태도 표현할 수 없다.

→ **`Long` 선택은 트레이드오프다.** `null` 표현력을 얻는 대신 캐시 함정을 떠안는다.

### 권한 검사에서의 실패 방향

```java
if (notice.getWriterId() == loginUser.getId()) { /* 수정 허용 */ }   // 잘못된 코드
```

- ID 127 까지: 캐시된 같은 객체라 정상 동작 → **초기 테스트를 그대로 통과한다**
- ID 128 부터: 값이 같아도 다른 객체 → 소유자 본인이 거부됨
- 실패 방향은 **fail-closed**(정당한 사용자 거부). 기능 장애이지 보안 사고는 아니다.

**진짜 위험한 fail-open 은 래퍼 타입 불일치다.**

```java
List<Long> blockedUserIds = ...;
int currentUserId = ...;
if (blockedUserIds.contains(currentUserId)) { throw new BlockedUserException(); }   // 항상 false
```

`Long.equals()` 의 첫 줄이 `instanceof Long` 이므로 `Integer` 를 넘기면 값이 같아도 무조건 `false`.
`contains()` 는 `Object` 를 받으므로 컴파일 경고조차 없다 → **차단 목록이 통째로 무력화된다.**

### 권장

```java
if (!Objects.equals(loginUser.getId(), notice.getWriterId())) {
    throw new AccessDeniedException();
}
```

> [!WARNING]
> **오답 코너**
> - **"`Long a=100L, b=100L` 이면 `a == b` 는 `false`"** — **`true`**. 캐시된 같은 객체를 공유한다.
> - **"`valueOf` 는 메모리 절약을 위해 int 로 변환한다"** — 타입은 끝까지 `Long`. **객체를 재사용**할 뿐이다.
> - **"캐시 범위는 -127~128"** — **-128 ~ 127** (부호 있는 8비트).
> - **"128 이상이면 변수에 값 대신 참조가 저장된다"** — 래퍼면 **항상** 참조가 저장된다.
>   달라지는 것은 그 참조가 같은 객체를 가리키느냐 다른 객체를 가리키느냐다.
> - **"-128~127 은 원시타입의 범위"** — `valueOf` 의 **캐시 범위**일 뿐이다.

## 복습 체크

- [ ] `Long a = 100L` 이 어떤 메서드 호출로 컴파일되는가?
- [ ] 왜 하필 -128~127 만 캐시하며, 그 숫자는 무엇의 범위인가?
- [ ] 128 미만에서 `==` 가 `true` 인 것을 어떻게 해석해야 하는가?
- [ ] `@Id` 를 `Long` 으로 두는 이유를 `isNew()` 와 연결해 설명할 수 있는가?
- [ ] `==` 로 소유자를 확인하면 실패 방향이 fail-open 인가 fail-closed 인가?
- [ ] `List<Long>.contains(int)` 가 왜 항상 `false` 인가?

## 관련

[[reference-vs-value-equality]] · [[equals-hashcode-contract]] · [[jpa-entity-equality]]
