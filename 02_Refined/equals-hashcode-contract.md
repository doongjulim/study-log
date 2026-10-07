---
type: refined
slug: equals-hashcode-contract
tags: [equals, hashCode, 규약, contract, 비둘기집원리, pigeonhole, 해시충돌, hash-collision, HashMap, HashSet, 버킷, bucket, 해시버킷, 가변키, mutable-key, 유령원소, 손실압축, 정확성, correctness, 성능, 상수hashCode, 분포, treeify, 재정의, override]
topic: equals 를 재정의하면 hashCode 도 재정의해야 하는 이유 — 규약은 한 방향뿐이고, 위반은 정확성 문제다
summary: 필수 규약은 "equals 가 true 면 hashCode 도 같다" 한 방향뿐이며 역방향은 int 의 유한성 때문에 지킬 수조차 없다. HashMap 은 hashCode 로 버킷을 정하고 equals 로 확정하므로 두 메서드가 같은 기준으로 판단해야 하고, 해시에 넣은 필드가 변하면 원소가 유령이 된다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Object.html#hashCode()
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/HashMap.html
updated: 2026-10-07
---

# equals / hashCode 규약

## 개념 정의

|  | 명제 | 지위 |
|---|---|---|
| **(가)** | `a.equals(b)` 가 `true` → `a.hashCode() == b.hashCode()` | **필수 규약** |
| (나) | `a.hashCode() == b.hashCode()` → `a.equals(b)` 가 `true` | **규약 아님. 지킬 수도 없음** |

`hashCode()` 의 반환은 `int` — 약 43억 개뿐인데 객체는 무한히 만들 수 있다.
비둘기집 원리상 **충돌은 버그가 아니라 설계상 필연**이다.
(나)가 규약이라면 표준 라이브러리인 `String` 이 이미 위반이다 — `"Aa".hashCode() == "BB".hashCode()` 는 `2112` 로 같지만 `equals` 는 `false`.

→ `hashCode()` 는 **손실 압축**이고, 그래서 `HashMap` 이 마지막에 `equals()` 로 한 번 더 확인하는 구조를 가질 수밖에 없다.
충돌이 없다면 그 단계 자체가 불필요했을 것이다.

> **규약 위반은 정확성(correctness) 문제, 해시 분포는 성능(performance) 문제.** 차원이 다르다.

## 작동 원리

### HashMap 탐색 3단계

```
1. hashCode()  → 버킷 배열의 인덱스를 결정      (후보를 빠르게 좁힘, O(1) 근사)
2. 해시 충돌 시 같은 칸에 여러 엔트리가 리스트/트리로 매달림
3. 그 칸 안에서 equals() 로 "진짜 그 key 인가" 최종 확정   (정확도)
```

`hashCode` 는 위치를 좁히고 `equals` 는 확정한다. **대체재가 아니라 역할 분담이다.**

### 한쪽만 재정의하면 각각 어디서 깨지는가

```java
Set<Member> set = new HashSet<>();
set.add(new Member(1L));
set.add(new Member(1L));   // 같은 값의 다른 객체
set.size();                // ?
```

| | 1단계(버킷) | 3단계(equals) | size |
|---|---|---|---|
| **equals 만 재정의** | ❌ `Object.hashCode`(객체 정체성) → 다른 버킷으로 흩어짐 | 호출조차 안 됨 | 2 |
| **hashCode 만 재정의** | ✅ 같은 버킷 도착 | ❌ `Object.equals` = `==` → `false` | 2 (같은 칸에 나란히) |

**핵심 통찰**: hashCode 만 재정의한 경우는 사실 **규약을 위반하지 않았다.**
`equals` 가 기본 구현이면 `true` 가 나오는 경우는 동일 객체뿐이고 그때는 해시도 같으니 (가)는 지켜진다.
**규약을 지켰는데도 원하는 동작은 얻지 못한 것이다.**
규약 준수는 최소 조건일 뿐, 진짜 요구사항은 **두 메서드가 같은 기준으로 판단할 것**이다.

### 가변 필드를 hashCode 에 넣으면 — 유령 원소

```java
@Override public int hashCode() { return Objects.hash(id); }   // id 가 나중에 채워진다면?
```

```
add 시점 hashCode : 31        ← Objects.hash(null)
contains (변경 전) : true
값 변경 후 hashCode: 32
contains (변경 후) : false     ← 방금 넣은 바로 그 객체인데!
size()            : 1         ← 분명히 안에 있음
순회로는 찾히는가  : true
```

`add()` 때 31번 버킷에 저장됐는데 해시가 32로 바뀌자 `contains()` 는 32번 칸을 뒤진다.
**`HashSet` 은 이미 넣은 원소를 재배치하지 않는다.**

결과적으로 이 원소는 **`size()` 에는 잡히고 순회로는 보이는데 `contains()`·`remove()` 로는 영영 못 찾는 유령**이 된다.

## 트레이드오프 / 한계

- **상수 `hashCode`(예: `getClass().hashCode()`)는 규약을 완벽히 지킨다.** 모든 해시가 같으니 (가)가 자동으로 참이다.
  대신 모든 원소가 한 버킷에 몰려 탐색이 **O(n) 선형**(= `ArrayList.contains()` 수준)이 된다.
  정확성을 사고 성능을 내주는 거래이며, 원소 수가 적으면 성립한다 ([[jpa-entity-equality]]).
- Java 8+ `HashMap` 은 한 버킷에 8개 이상 쌓이면 트리로 전환해 O(log n) 이 되지만 `Comparable` 구현이 전제다 — 해시가 전부 같으면 트리도 hash 로는 가를 수 없기 때문 (체이닝·treeify·resize 전체 구조는 [[hashmap-internals]]).
- 해시 분포가 나쁜 것은 느려질 뿐 **틀리지는 않는다.** 반대로 규약을 어기면 데이터를 잃는다.

> [!WARNING]
> **오답 코너**
> - **"필수 규약은 (나) — 해시가 같으면 equals 도 true"** — **(가)만 필수**다.
>   (나)는 `String` 조차 못 지키며, `int` 가 유한해 **지킬 수가 없다.**
> - **"상수 `hashCode` 는 규약 위반"** — **완벽하게 준수한다.** 위반하려야 할 수가 없다.
> - **"`hashCode` 를 빠뜨리면 같은 버킷에서 충돌 처리 중 어긋난다"** — **1단계**에서 어긋난다.
>   다른 버킷으로 흩어져 `equals()` 가 호출조차 되지 않는다.
> - **"`HashSet.contains()` 는 원소 수만큼 비교한다"** — O(1) 근사다. 원소 수만큼 비교하는 건 `ArrayList.contains()`.
> - **"해시 충돌로 인한 선형 탐색 = N+1 문제"** — 다른 축이다.
>   **N+1 은 쿼리 횟수, O(n) 은 비교 횟수.** 원인도 해법도 다르다 ([[n-plus-1-problem]]).

## 복습 체크

- [ ] 두 방향의 명제 중 어느 쪽이 필수이고, 나머지는 왜 지킬 수 없는가?
- [ ] `HashMap` 탐색 3단계에서 `hashCode` 와 `equals` 의 역할을 각각 말할 수 있는가?
- [ ] `equals` 만 / `hashCode` 만 재정의했을 때 각각 몇 단계에서 어긋나는가?
- [ ] `hashCode` 만 재정의한 경우가 "규약 위반이 아닌데도 실패"인 이유는?
- [ ] 유령 원소가 만들어지는 과정을 버킷 번호로 설명할 수 있는가?
- [ ] 규약 위반과 해시 분포 불량은 각각 무엇을 잃는 문제인가?

## 관련

[[hashmap-internals]] · [[reference-vs-value-equality]] · [[jpa-entity-equality]] · [[wrapper-cache-autoboxing]] · [[string-immutability]] · [[deterministic-hash-for-lookup]]
