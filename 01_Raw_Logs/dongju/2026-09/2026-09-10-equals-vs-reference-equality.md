---
date: 2026-09-10
type: raw
author: dongju
tags: [==, equals, 동일성, identity, 동등성, equality, 참조 비교, reference equality, 값 비교, value equality, 오토박싱, autoboxing, 래퍼 클래스, wrapper class, Long, Integer, valueOf, 정수 캐시, Integer cache, LongCache, hashCode, HashMap, 버킷, bucket, 해시 충돌, hash collision, Objects.equals, NPE, NullPointerException, 소유자 확인, 권한 검사, authorization, fail-open, fail-closed, JPA, 엔티티 식별자, entity id, isNew, 원시타입, primitive]
topic: == 와 equals() 의 차이, 오토박싱 캐시가 만드는 함정, 그리고 소유자 확인을 equals 로 해야 하는 이유
summary: == 는 "변수 상자에 담긴 값 자체"를 비교할 뿐이며, 참조타입에서는 그 값이 참조이므로 결과적으로 동일 객체 여부를 묻게 된다. JPA 엔티티의 @Id 가 Long 인 이상 -128~127 밖에서는 같은 값이라도 서로 다른 객체가 되어 == 소유자 확인이 128번째 회원부터 조용히 깨진다.
source: session
distilled: true
distilled_at: 2026-09-10
---

## 배운 개념

### 1. `==` 의 정의는 하나뿐이다

`==` 는 **변수라는 상자에 저장된 값 자체를 비교한다.** 그 값이 무엇을 의미하는지는 전혀 신경 쓰지 않는다.

| 변수 종류 | 상자에 든 것 | `==` 가 결과적으로 묻는 것 |
|---|---|---|
| 원시타입 | 데이터 그 자체 (비트열) | 값이 같은가 |
| 참조타입 | 객체를 가리키는 참조 | **같은 객체인가** |

동작은 처음부터 끝까지 한 가지인데, **담긴 것이 달라서 의미가 달라 보일 뿐**이다.
"`==` 는 주소를 비교한다"는 설명은 참조타입이라는 특수 케이스를 전체 정의로 착각한 것이다.
`long p = 1000L; long q = 1000L;` 에서 `p == q` 가 `true` 인 것이 그 반례다 — 여기엔 힙도 객체도 주소도 없다.

### 2. `equals()` 와 `hashCode()` 는 대체재가 아니라 역할 분담이다

- `Object.equals()` 의 기본 구현은 `==` 와 동일하다. 값 비교를 원하면 각 클래스가 오버라이드해야 한다.
- `String.equals()` 는 문자를 하나씩 비교한다. **해시값 비교가 아니다.**
  - 반례: `"Aa".hashCode() == "BB".hashCode()` → 둘 다 `2112` 로 같지만 `"Aa".equals("BB")` 는 `false`.

해시 비교가 실제로 등장하는 자리는 `equals()` 내부가 아니라 **`HashMap` 의 버킷 탐색**이다.

```
1. hashCode()  → 버킷 배열의 인덱스를 결정     (후보를 빠르게 좁힘, O(1) 근사)
2. 해시 충돌 시 같은 칸에 여러 엔트리가 리스트/트리로 매달림
3. 그 칸 안에서 equals() 로 "진짜 그 key 인가" 최종 확정  (정확도)
```

→ `equals()` 만 오버라이드하고 `hashCode()` 를 빠뜨리면 `Object.hashCode()`(객체 정체성 기반)가 쓰여
**1단계에서 아예 다른 버킷으로 흩어진다.** 그 결과 `equals()` 는 **호출조차 되지 않고** `get()` 이 `null` 을 반환한다.
"equals 를 재정의하면 hashCode 도 반드시 재정의하라"는 규약의 실질적 이유가 이것이다.

### 3. 오토박싱과 `Long` / `Integer` 캐시

`Long a = 100L;` 은 컴파일러가 **`Long.valueOf(100L)`** 호출로 바꾼다. `new Long(100L)` 이 아니다.

```java
public static Long valueOf(long l) {
    if (l >= -128 && l <= 127) return LongCache.cache[(int)l + 128]; // 미리 만들어둔 객체 재사용
    return new Long(l);                                             // 범위 밖은 새 객체
}
```

JVM 이 클래스 로딩 시점에 -128~127 에 해당하는 객체 256개를 미리 만들어 배열에 넣어둔다.
0, 1, 200 같은 작은 정수는 압도적으로 자주 쓰이므로 매번 `new` 하면 힙에 쓰레기가 쌓이고 GC 부담이 커진다.
**자주 쓰이는 소수의 값만 미리 만들어 공유하는 메모리·GC 최적화**다.
(-128~127 은 부호 있는 8비트, 즉 `byte` 의 표현 범위와 같다.)

실제 실행 결과:

```
Long a = 100L, b = 100L;
a 객체 정체성 : 148912029
b 객체 정체성 : 148912029   ← 완전히 동일. 두 개가 아니라 하나.
a == b : true

Long c = 1000L, d = 1000L;
c == d      : false
c.equals(d) : true

Long x = new Long(100L), y = new Long(100L);   // 캐시 강제 우회
x 객체 정체성 : 874217650
y 객체 정체성 : 1436664465  ← 서로 다른 객체
x == y      : false
x.equals(y) : true
```

**가장 중요한 해석**: 128 미만에서 `==` 가 `true` 인 것은 "`==` 가 값 비교를 해준 것"이 아니라
**"우연히 같은 객체였던 것"**이다. `==` 는 두 경우 모두 똑같이 동작했고,
달라진 것은 `valueOf` 가 객체를 재사용했는지 여부뿐이다.

### 4. `Objects.equals()` — `==` 와 `equals()` 의 협력

```java
public static boolean equals(Object a, Object b) {
    return (a == b) || (a != null && a.equals(b));
}
```

- `==` 로 먼저 거른다 → 둘 다 `null` 이면 `a == b` 가 `true` 라 바로 통과.
- `a != null` 이 앞에 있어 `a.equals()` 는 `a` 가 확실히 `null` 이 아닐 때만 호출된다 → NPE 차단.

이 유틸리티를 쓰지 않는다면 **`null` 이 아님이 보장된 쪽을 왼쪽에** 둔다.
소유자 확인이라면 인증을 통과한 로그인 사용자 ID 가 그 쪽이다.

### 5. 소유자 확인에 적용 — study-board

`@Id` 를 `Long`(래퍼)으로 두는 이유는 `==` 가 불편해서가 아니라
**"값 없음"이라는 상태가 도메인에 실재하기 때문**이다.

- 원시타입 `long` 은 `null` 을 담지 못하므로 "아직 채번 전"을 `0` 으로밖에 표현할 수 없다.
- Spring Data JPA 의 `AbstractEntityInformation.isNew()` 는 ID 가 `null` 이거나
  **원시 숫자 타입이면서 값이 `0`** 일 때 새 엔티티로 판단한다.
- 그래서 `0` 은 **"미할당"인지 "진짜 0번 행"인지 구분 불가능**해진다. 시퀀스가 0부터 시작하는 DB 를 쓰면
  이미 저장된 엔티티가 계속 새 것으로 오인된다.
- 익명 게시글처럼 **작성자가 없는 상태**도 표현할 수 없다.

즉 `Long` 선택은 **트레이드오프**다. `null` 표현력을 얻는 대신 오토박싱 캐시 함정을 떠안고,
그래서 `equals()` 가 필요해진다.

```java
// 잘못된 코드
if (notice.getWriterId() == loginUser.getId()) { ... }
```

- ID 127 까지: 캐시된 같은 객체라 정상 동작 → **초기 테스트를 그대로 통과한다**
- ID 128 부터: 값이 같아도 서로 다른 객체 → 소유자 본인도 거부됨
- **데이터가 쌓여야만 드러나는 시간 지연형 버그**

이 버그의 실패 방향은 **fail-closed**(정당한 사용자를 거부)다. 짜증나는 기능 장애이지 보안 사고는 아니다.

```java
// 최종 권장
if (!Objects.equals(loginUser.getId(), notice.getWriterId())) {
    throw new AccessDeniedException();
}
```

### 6. 진짜 위험한 fail-open — 래퍼 타입 불일치

```java
List<Long> blockedUserIds = ...;   // 차단된 회원 ID 목록
int currentUserId = ...;           // 어딘가에서 int 로 넘어옴

if (blockedUserIds.contains(currentUserId)) {   // 항상 false
    throw new BlockedUserException();
}
```

`Long.equals()` 의 첫 줄이 `if (obj instanceof Long)` 이므로 `Integer` 를 넘기면 **값이 같아도 무조건 `false`** 다.
`contains()` 는 `Object` 를 받으므로 컴파일 경고조차 없다. → **차단 목록이 통째로 무력화된다(fail-open).**
소유자 확인이 fail-closed 로 깨지는 것보다 훨씬 위험한 방향.

## 오답·헷갈린 점

| 처음 생각 (오개념) | 정정 |
|---|---|
| `==` 는 힙의 저장 위치를 비교한다 | **상자에 담긴 값 자체**를 비교한다. 원시타입은 힙이 없어도 `==` 가 동작한다 (`long p==q` → `true`) |
| `equals()` 는 값의 해시값을 비교한다 | 값 자체를 비교한다. `"Aa"` 와 `"BB"` 는 해시가 같지만 `equals()` 는 `false` |
| 해시 비교는 `equals()` 안에서 일어난다 | `HashMap` 의 **버킷 인덱스 결정** 단계에서 일어난다. `hashCode`=후보 좁히기, `equals`=최종 확정 |
| `hashCode()` 미구현 시 2단계(충돌 리스트)에서 어긋난다 | **1단계**에서 어긋난다. 다른 버킷으로 흩어져 `equals()` 가 호출조차 안 된다 |
| `Long a=100L, b=100L` → `a==b` 는 `false` | **`true`.** 캐시된 **같은 객체 하나**를 공유하기 때문 |
| `Long.valueOf` 는 메모리 절약을 위해 int 로 변환한다 | 타입은 끝까지 `Long`. **객체를 재사용**할 뿐. (절약이 목적이라는 방향은 맞았음) |
| 캐시 범위는 -127~128 | **-128 ~ 127.** 부호 있는 8비트(`byte`) 범위와 같다 |
| 128 이상이면 변수에 값 대신 참조가 저장된다 | `Long` 이면 **항상** 참조가 저장된다. 달라지는 건 그 참조가 **같은 객체를 가리키느냐 다른 객체를 가리키느냐** |
| -128~127 은 "원시변수의 범위" | 원시타입 `long` 의 범위는 약 ±922경. -128~127 은 **`valueOf` 의 캐시 범위**일 뿐 |
| NPE 회피 유틸리티는 `Optional` | **`java.util.Objects.equals(a, b)`** |
| `long id` 가 `0` 이면 "0번 행" | Spring Data 는 **"아직 저장 안 된 새 엔티티"**로 판단한다 |

## Q&A

**Q. `==` 를 힙·주소라는 단어 없이 한 문장으로 정의하면?**
변수라는 상자에 저장된 값 자체를 비교한다. 원시타입이면 그 값이 곧 데이터고, 참조타입이면 그 값이 객체를 가리키는 참조다.

**Q. `Long c = 1000L; Long d = 1000L;` 에서 `c == d` 가 `false` 인 이유는?**
`Long` 은 객체 타입이라 변수에 참조가 담긴다. 1000 은 `valueOf` 캐시 범위(-128~127) 밖이므로 각각 `new Long` 으로 새 객체가 만들어지고, 두 변수에 서로 다른 참조가 담긴다. `==` 는 그 참조를 비교하므로 값이 같아도 `false` 다.

**Q. `equals()` 만 오버라이드하고 `hashCode()` 를 안 하면?**
`Object.hashCode()`(객체 정체성 기반)가 쓰여 값이 같은 두 객체가 서로 다른 버킷 인덱스로 배정된다. `equals()` 가 호출되는 3단계까지 도달하지 못하고 `get()` 이 `null` 을 반환한다.

**Q. `==` 에는 없고 `equals()` 에만 있는 위험은?**
`null` 참조에서의 NPE. `==` 는 `null` 이어도 조용히 `false` 를 내지만 `null.equals(...)` 는 터진다. `Objects.equals()` 로 회피하거나, `null` 이 아님이 보장된 쪽을 왼쪽에 둔다.

**Q. `@Id` 를 원시타입 `long` 으로 하면 `==` 가 항상 안전하지 않나?**
`long` 은 "값 없음"을 표현하지 못한다. 미할당 상태가 `0` 이 되는데, Spring Data 는 원시 숫자 `0` 을 "새 엔티티"로 판단하므로 실제 0번 행과 구분이 불가능해진다. 익명 작성자 같은 상태도 표현할 수 없다.

**Q. 면접 30초 답변 (소유자 확인을 왜 equals 로 하나요?)**
JPA 엔티티의 `@Id` 는 "아직 채번되지 않음"을 표현해야 해서 원시타입 `long` 이 아니라 래퍼 `Long` 을 씁니다. `Long` 변수에는 항상 참조가 담기고, `==` 는 그 상자에 담긴 값(참조)을 비교하므로 결국 "같은 객체인가"를 묻게 됩니다. 그런데 `Long.valueOf` 는 -128~127 만 캐시된 객체를 재사용하므로, ID 가 127 이하일 때는 우연히 같은 객체라 `==` 가 통과하다가 128번째 회원부터 조용히 실패합니다. 초기 테스트는 통과하고 운영에서 터지는 버그죠. 그래서 값 비교인 `equals()` 를 쓰되, `null` 안전까지 확보하려고 `Objects.equals()` 를 사용합니다.

## 다음 학습

- JPA 엔티티 자체에 `equals`/`hashCode` 를 오버라이드할 때의 함정 — 지연 로딩 프록시, `getClass()` vs `instanceof`, ID 기반 vs 비즈니스 키 기반
- `Integer` 캐시 범위를 `-XX:AutoBoxCacheMax` 로 조정할 수 있다는 점과 그것이 왜 위험한지 (JLS 가 보장하는 하한은 127)
- `record` 가 자동 생성하는 `equals`/`hashCode` 의 동작과 한계
- `String` 의 `==` 함정 — 문자열 상수 풀(intern)과 `new String()`, 컴파일 타임 상수 폴딩
