---
date: 2026-09-10
type: raw
author: dongju
tags: [equals, hashCode, 규약, contract, 비둘기집 원리, pigeonhole, 해시 충돌, hash collision, HashSet, HashMap, 버킷, bucket, 가변 필드, mutable key, 유령 원소, StackOverflowError, Error vs Exception, 무한 재귀, infinite recursion, Lombok, EqualsAndHashCode, JPA 엔티티, entity equals, 프록시, proxy, LazyInitializationException, N+1, getClass, instanceof, 대칭성, symmetry, 1차 캐시, first-level cache, 동일성 보장, 상수 hashCode, GeneratedValue]
topic: equals 를 재정의하면 hashCode 도 재정의해야 하는 이유와, JPA 엔티티에서 두 메서드를 어떻게 구현해야 하는가
summary: 필수 규약은 "equals 가 true 면 hashCode 도 같다" 한 방향뿐이며 역방향은 int 의 유한성 때문에 지킬 수조차 없다. JPA 엔티티에 Lombok 전체 필드 @EqualsAndHashCode 를 붙이면 무한 재귀·프록시 예외·N+1 세 지뢰를 동시에 밟고, id 는 save 시점에 변하는 가변 필드라 hashCode 에 넣을 수 없어 결국 상수 hashCode + instanceof + id 기반 equals 가 답이 된다.
source: session
distilled: true
distilled_at: 2026-09-10
---

## 배운 개념

### 1. 규약은 한 방향뿐이다

|  | 명제 | 지위 |
|---|---|---|
| **(가)** | `a.equals(b)` 가 `true` → `a.hashCode() == b.hashCode()` | **필수 규약** |
| (나) | `a.hashCode() == b.hashCode()` → `a.equals(b)` 가 `true` | **규약 아님. 지킬 수도 없음** |

`hashCode()` 의 반환 타입은 `int` — 약 43억 개뿐인데 객체는 무한히 만들 수 있다.
비둘기집 원리상 **충돌은 버그가 아니라 설계상 필연**이다.
(나)가 규약이라면 자바 표준 라이브러리인 `String` 이 이미 규약 위반이다 — `"Aa".hashCode() == "BB".hashCode()` 는 `2112` 로 같지만 `equals` 는 `false`.

→ `hashCode()` 는 **손실 압축**이고, 그래서 `HashMap` 이 3단계에서 `equals()` 로 한 번 더 확인하는 구조를 가질 수밖에 없다. 충돌이 없다면 3단계 자체가 불필요했을 것이다.

→ **규약 위반은 정확성(correctness) 문제, 해시 분포는 성능(performance) 문제.** 차원이 다르다.

### 2. 한쪽만 재정의하면 각각 어디서 깨지는가

```java
class Member { Long id; }

Set<Member> set = new HashSet<>();
set.add(new Member(1L));
set.add(new Member(1L));   // 같은 값의 다른 객체
System.out.println(set.size());
```

| | 1단계 (버킷 인덱스) | 3단계 (equals) | size |
|---|---|---|---|
| **equals 만 재정의** | ❌ `Object.hashCode`(객체 정체성) → 다른 버킷으로 흩어짐 | 호출조차 안 됨 | 2 |
| **hashCode 만 재정의** | ✅ 같은 버킷 도착 | ❌ `Object.equals` = `==` → `false` | 2 (같은 칸에 둘이 나란히) |

**핵심 통찰**: hashCode 만 재정의한 경우는 사실 **규약을 위반하지 않았다.**
`equals` 가 `Object` 기본 구현이라 `true` 가 나오는 경우는 동일 객체뿐이고, 그때는 `hashCode` 도 당연히 같으니 (가)는 지켜진다.
**규약을 지켰는데도 원하는 동작은 얻지 못한 것이다.** 규약 준수는 최소 조건일 뿐, 진짜 요구사항은 **두 메서드가 같은 기준으로 판단할 것**이다.

### 3. Lombok `@EqualsAndHashCode`(전체 필드)가 JPA 엔티티에서 밟는 지뢰 3개

옵션 없이 붙이면 **모든 필드**가 비교 대상에 들어간다.

#### ① 무한 재귀 → `StackOverflowError`

```java
@Entity @EqualsAndHashCode
class Member { @Id Long id; String name; @OneToMany(mappedBy="writer") List<Notice> notices; }

@Entity @EqualsAndHashCode
class Notice { @Id Long id; String title; @ManyToOne Member writer; }
```

```
member1.equals(member2)
  └─ ① id, name … 그리고 notices 리스트 비교
       └─ ② List.equals() 는 원소끼리 equals 호출 → notice1.equals(notice2)
            └─ ③ Notice.equals 는 id, title … 그리고 writer 필드를 비교
                 └─ ④ writer 는 Member 타입 → member1.equals(member2)
                      └─ ⑤ ②로 복귀 — 끝나지 않는다
```

`while(true)` 와 달리 **재귀는 호출마다 스택 프레임을 쌓으므로** 영원히 도는 게 아니라 몇 초 안에 죽는다.

```
Throwable
├── Exception   ← 애플리케이션이 대응할 수 있는 상황 (복구 시도 가능)
└── Error       ← JVM 수준에서 무너진 상황 (복구 대상이 아님)
    ├── StackOverflowError
    └── OutOfMemoryError
```

**실무에서 이 구분이 치명적인 이유**: `Error` 는 `Exception` 의 하위가 아니므로

```java
@ExceptionHandler(Exception.class)   // StackOverflowError 는 여기 안 걸림
```

전역 예외 핸들러를 그대로 통과해 톰캣까지 올라간다. 로그에는 수천 줄의 동일 프레임이 반복되고 사용자에겐 정제되지 않은 500이 나간다.
게다가 스택이 이미 무너진 상태라 `catch` 안에서 뭘 하려 해도 다시 터질 수 있다. **고칠 대상은 예외 처리가 아니라 `equals` 구현 자체.**

#### ② 프록시 → `LazyInitializationException`

`@ManyToOne(fetch = LAZY)` 필드에는 진짜 엔티티가 아니라 **프록시**가 들어있다.
`equals` 가 그 실제 값을 요구하면 프록시는 자기를 만든 Session 에 위임하는데([[lazy-loading-proxy]] 3단계),
`open-in-view=false` 로 그 Session 이 이미 닫혔다면 예외가 터진다.

**오늘 새로 얻은 것은 "어디서 터지느냐"다.**
지금까지 이 예외는 "컨트롤러·뷰에서 지연 연관을 건드릴 때 나는 것"으로 알고 있었다.
그런데 `equals`/`hashCode` 안에 지연 연관이 들어가면 **`set.add()`, `list.contains()`, `assertEquals()` 같은 평범한 연산이 예외 발생 지점**이 된다.
스택 트레이스가 `HashSet.add` 로 찍히니 진단이 훨씬 어렵다. **`equals` 는 아무도 터질 거라 예상하지 않는 메서드**이기 때문에 특히 위험하다.

#### ③ N+1

```java
Set<Notice> set = new HashSet<>();
for (Notice n : notices) set.add(n);   // 1000개
```

`add()` → `hashCode()` 1000회 → 각각 `writer` 프록시 초기화 → **`SELECT` 1000번.**
쿼리를 유발할 만한 코드가 하나도 안 보이는데 쿼리가 쏟아진다.
`join fetch` 로 아무리 최적화해도 `equals`/`hashCode` 가 뒤에서 다시 프록시를 깨우면 소용없다. ([[n-plus-1-problem]])

### 4. 가변 필드를 `hashCode` 에 넣으면 — 유령 원소

```java
@Override public int hashCode() { return Objects.hash(id); }   // id 는 가변이다
```

```java
Notice notice = new Notice("제목");   // 저장 전 → id == null
Set<Notice> set = new HashSet<>();
set.add(notice);
noticeRepository.save(notice);        // 여기서 id 가 채워짐
set.contains(notice);                 // ???
```

실제 실행 결과:

```
add 시점 hashCode : 31        ← Objects.hash(null)
contains (저장 전) : true
save 후 hashCode  : 32        ← Objects.hash(1L)
contains (저장 후) : false     ← 방금 넣은 바로 그 객체인데!
set.size()        : 1         ← 분명히 안에 있음
순회로는 찾히는가 : true       ← for 문으로는 찾힘
```

`add()` 시점엔 해시가 `31` 이라 **31번 버킷에 저장**됐다. `save()` 가 `id` 를 채우자 해시가 `32` 로 바뀌었다.
그런데 **`HashSet` 은 이미 넣은 원소를 재배치하지 않는다.** 객체는 31번 칸에 있는데 `contains()` 는 32번 칸을 뒤진다 — 1단계에서 엉뚱한 칸으로 간다.

결과적으로 이 원소는 **`size()` 에는 잡히고 순회로는 보이는데 `contains()` 로는 영영 못 찾는 유령**이 된다. `remove()` 도 실패하니 빼낼 수도 없다.

**`@GeneratedValue` 를 쓰는 한 엔티티의 `id` 는 `save()` 시점에 채워지는 대표적인 가변 필드다.** 피할 수 없다.

### 5. `getClass()` vs `instanceof`

프록시는 원본 클래스를 **상속한** 껍데기(`Member$HibernateProxy$abc123`)다.

```java
Member real  = memberRepository.findById(1L).get();
Member proxy = notice.getWriter();

// getClass() != o.getClass()  →  다른 클래스로 판정 → id 비교까지 가지도 못함
// o instanceof Member         →  프록시는 하위 타입이므로 통과
```

대가: `instanceof` 는 **대칭성(symmetry) 규약이 깨질 여지**를 만든다. `A.equals(B)` 와 `B.equals(A)` 가 달라질 수 있다.
JPA 에서는 프록시가 일상이므로 이 트레이드오프를 받아들이는 것이 사실상 표준.
(Hibernate 6 라면 `Hibernate.getClass(o)` 로 프록시를 벗겨내는 방법도 있다.)

### 6. 권장 구현과 상수 hashCode 가 성립하는 이유

```java
@Entity
class Notice {
    @Id @GeneratedValue Long id;
    String title;
    @ManyToOne(fetch = LAZY) Member writer;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Notice)) return false;      // 프록시 호환
        return id != null && Objects.equals(id, ((Notice) o).id);
    }

    @Override
    public int hashCode() {
        return getClass().hashCode();                  // 절대 변하지 않는 상수
    }
}
```

**`hashCode` 에 넣을 수 있는 필드가 하나도 없다.**

- 연관 필드 → 무한 재귀·프록시 예외·N+1
- `title` 같은 일반 필드 → 수정되면 해시가 바뀜
- `id` → `save()` 시점에 `null` 에서 값으로 바뀜

**상수는 규약 (가)를 어떤 경우에도 자동으로 만족시킨다.** 모든 해시가 같으니 위반하려야 할 수가 없다 → 정확성 완전 확보.

대가는 **O(n) 선형 탐색**이다. 모든 원소가 같은 버킷 하나에 리스트로 매달리고, 3단계에서 그 리스트를 처음부터 순회한다. 해시 자료구조를 쓰면서 해시의 이점을 통째로 반납한 셈으로, `ArrayList.contains()` 와 같은 성능이다.

**그럼에도 감당 가능한 이유 두 가지:**

1. **1차 캐시 동일성 보장** — 하나의 영속성 컨텍스트 안에서 같은 ID 의 엔티티는 인스턴스가 딱 하나다. 그래서 대부분의 비교는 `equals` 첫 줄 `if (this == o) return true;` 에서 즉시 끝난다.
2. **컬렉션 규모** — 한 회원이 쓴 공지, 한 스터디의 참여자 같은 엔티티 컬렉션은 보통 수십~수백 개다. O(n) 이지만 n 이 작아 실질적으로 문제가 되지 않는다.

즉 **성능을 내주고 정확성을 산 거래**다. 정확성이 깨지면 데이터를 잃지만(유령 원소), 성능은 n 이 작으면 감당된다.

## 오답·헷갈린 점

| 처음 생각 (오개념) | 정정 |
|---|---|
| 필수 규약은 (나) "해시가 같으면 equals 도 true" | **(가)만 필수.** (나)는 `String` 조차 못 지킨다 (`"Aa"`/`"BB"` 충돌). `int` 가 유한해서 **지킬 수가 없다** |
| 상수 `hashCode` 는 규약 위반이다 | **완벽하게 준수한다.** 모든 해시가 같으니 (가)가 자동으로 참. 위반하려야 할 수가 없다 |
| `StackOverflowException` | **`StackOverflowError`.** `Exception` 이 아니라 `Error` → `@ExceptionHandler(Exception.class)` 로 못 잡는다 |
| 무한 재귀는 "무한 루프"다 | 재귀는 **스택 프레임을 쌓으므로** 영원히 돌지 않고 몇 초 안에 `StackOverflowError` 로 죽는다 |
| `HashSet.contains()` 는 원소 수만큼 비교한다 | O(1) 근사. 버킷으로 곧장 점프해 소수와만 비교한다. **1000번 비교하는 건 `ArrayList.contains()`** |
| 해시 충돌로 인한 선형 탐색 = N+1 문제 | **다른 축이다.** N+1 은 **쿼리 횟수**, O(n) 은 **비교 횟수**. 원인도 해법도 다르며 따로 또는 동시에 발생한다 |
| `getClass()` 프록시 문제의 해법은 `getId()` | **`instanceof`.** 문제는 비교 대상이 아니라 그 앞의 **타입 체크 관문**이다. `getClass()` 에서 이미 `return false` 로 빠져나간다 |

## Q&A

**Q. 왜 (나) 방향은 규약이 될 수 없나?**
`hashCode()` 의 반환은 `int` 로 약 43억 개뿐인데 객체는 무한히 만들 수 있다. 비둘기집 원리상 서로 다른 객체가 같은 해시를 갖는 일은 필연이다. 즉 (나)는 지키지 않는 게 아니라 지킬 수가 없다.

**Q. `hashCode` 만 재정의하면 규약 위반인가?**
아니다. `equals` 가 `Object` 기본 구현이면 `true` 가 나오는 경우는 동일 객체뿐이고 그때 해시도 같으므로 (가)는 지켜진다. 그런데도 `HashSet` 은 중복을 걸러내지 못한다 — 규약 준수는 최소 조건이지 충분 조건이 아니다.

**Q. `equals` 에 지연 연관을 넣으면 왜 특히 위험한가?**
`equals` 는 아무도 예외를 예상하지 않는 메서드라서다. `set.add()`, `list.contains()`, `assertEquals()` 안에서 조용히 호출되므로, 예외와 N+1 쿼리가 전혀 예상 못 한 지점에서 튀어나오고 스택 트레이스도 원인을 가린다.

**Q. `id` 를 `hashCode` 에 넣으면 안 되는 이유는?**
`@GeneratedValue` 라면 `save()` 시점에 `null` → 값으로 바뀐다. `HashSet` 은 원소를 재배치하지 않으므로, 저장 전에 넣어둔 객체는 저장 후 해시가 달라져 다른 버킷을 뒤지게 되고 `contains()`/`remove()` 로 영영 찾을 수 없는 유령이 된다.

**Q. 상수 `hashCode` 의 성능 저하는 왜 감당 가능한가?**
1차 캐시 동일성 보장으로 같은 ID 의 인스턴스는 컨텍스트당 하나뿐이라 대부분 `this == o` 에서 끝나고, 엔티티 컬렉션 자체가 수십~수백 개 규모라 O(n) 의 n 이 작다.

## 다음 학습

- `record` 가 자동 생성하는 `equals`/`hashCode` — 전체 필드 기반이라 엔티티에는 여전히 부적합한 이유
- 비즈니스 키(natural key) 기반 `equals` — 이메일·사업자번호처럼 불변인 값이 있다면 상수 `hashCode` 보다 나은 선택인가
- Java 8+ `HashMap` 의 treeify — 한 버킷에 8개 이상 쌓이면 트리로 전환되어 O(log n) 이 되지만 `Comparable` 구현이 전제
- study-board 실제 조치: `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` + `id` 에만 `@Include` 로 바꿀지, 직접 구현으로 교체할지
