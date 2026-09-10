---
type: refined
slug: jpa-entity-equality
tags: [JPA엔티티, entity-equals, equals, hashCode, Lombok, EqualsAndHashCode, 무한재귀, StackOverflowError, 양방향연관, 프록시, proxy, LazyInitializationException, getClass, instanceof, 대칭성, symmetry, 상수hashCode, 1차캐시, first-level-cache, 동일성보장, GeneratedValue, 가변식별자, N+1]
topic: JPA 엔티티에서 equals/hashCode 를 어떻게 구현해야 하는가 — 넣을 수 있는 필드가 하나도 없다는 결론에 이르는 과정
summary: Lombok 전체 필드 @EqualsAndHashCode 는 무한 재귀·프록시 예외·N+1 세 지뢰를 동시에 밟고, id 조차 save 시점에 변하는 가변 필드다. 결국 instanceof + id 기반 equals 와 절대 변하지 않는 상수 hashCode 가 답이 된다.
contributors: [dongju]
source_refs:
  - https://docs.jboss.org/hibernate/orm/6.4/userguide/html_single/Hibernate_User_Guide.html#entity-pojo-equalshashcode
updated: 2026-09-10
---

# JPA 엔티티의 equals / hashCode

## 개념 정의

일반 객체의 규약([[equals-hashcode-contract]])은 그대로 적용되지만, 엔티티에는 세 가지 특수 사정이 겹친다.

1. **연관 필드**가 있다 — 양방향이면 서로를 참조한다
2. 그 연관은 **프록시**일 수 있다 ([[lazy-loading-proxy]])
3. **식별자가 `save()` 시점에 채워진다** — `@GeneratedValue`

이 셋 때문에 "모든 필드를 비교한다"는 상식적인 구현이 정확히 실패한다.

## 작동 원리

### Lombok `@EqualsAndHashCode`(전체 필드)가 밟는 지뢰 3개

#### ① 무한 재귀 → `StackOverflowError`

```java
@Entity @EqualsAndHashCode
class Member { @Id Long id; String name; @OneToMany(mappedBy="writer") List<Notice> notices; }

@Entity @EqualsAndHashCode
class Notice { @Id Long id; String title; @ManyToOne Member writer; }
```

```
member1.equals(member2)
  └─ ① notices 리스트 비교
       └─ ② List.equals() 는 원소끼리 equals → notice1.equals(notice2)
            └─ ③ Notice.equals 는 writer 필드를 비교
                 └─ ④ member1.equals(member2)  ← ①로 복귀. 끝나지 않는다
```

`while(true)` 와 달리 **재귀는 호출마다 스택 프레임을 쌓으므로** 몇 초 안에 `StackOverflowError` 로 죽는다.
`Error` 는 `Exception` 의 하위가 아니라 `@ExceptionHandler(Exception.class)` 로 잡히지 않는다 ([[jvm-stack-and-heap]]).

#### ② 프록시 → `LazyInitializationException`

`@ManyToOne(fetch = LAZY)` 필드에는 프록시가 들어있다. `equals` 가 그 값을 요구하면
프록시는 자기를 만든 Session 에 위임하는데, 컨텍스트가 닫혔다면 예외가 터진다.

**중요한 것은 "어디서 터지느냐"다.** `equals`/`hashCode` 안에 지연 연관이 들어가면
`set.add()`, `list.contains()`, `assertEquals()` 같은 **평범한 컬렉션 연산이 예외 발생 지점**이 된다.
스택 트레이스가 `HashSet.add` 로 찍혀 진단이 훨씬 어렵다.

#### ③ N+1

`set.add()` 1000회 → `hashCode()` 1000회 → 각각 프록시 초기화 → **`SELECT` 1000번.**
쿼리를 유발할 코드가 안 보이는데 쿼리가 쏟아진다. `join fetch` 로 최적화해도
`equals`/`hashCode` 가 뒤에서 다시 프록시를 깨우면 소용없다 ([[n-plus-1-problem]]).

### `getClass()` vs `instanceof`

프록시는 원본 클래스를 **상속한** 껍데기(`Member$HibernateProxy$abc123`)다.

| | 원본 vs 프록시 |
|---|---|
| `getClass() != o.getClass()` | **다른 클래스로 판정** → id 비교까지 가지도 못함 |
| `o instanceof Member` | 프록시는 하위 타입이므로 **통과** |

대가: `instanceof` 는 대칭성(symmetry) 규약이 깨질 여지를 만든다.
JPA 에서는 프록시가 일상이므로 이 트레이드오프를 받아들이는 것이 사실상 표준이다.
(Hibernate 6 의 `Hibernate.getClass(o)` 로 프록시를 벗겨내는 방법도 있다.)

### 권장 구현

```java
@Entity
class Notice {
    @Id @GeneratedValue Long id;

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

## 트레이드오프 / 한계

**`hashCode` 에 넣을 수 있는 필드가 하나도 없다.**

| 후보 | 탈락 이유 |
|---|---|
| 연관 필드 | 무한 재귀 · 프록시 예외 · N+1 |
| `title` 같은 일반 필드 | 수정되면 해시가 바뀌어 유령 원소가 된다 |
| `id` | `save()` 시점에 `null` → 값으로 바뀐다 (가변) |

그래서 **상수 `hashCode`** 가 남는다. 규약 (가)를 어떤 경우에도 자동으로 만족시키므로 정확성은 완전히 확보된다.
대가는 O(n) 선형 탐색이지만 감당 가능한 이유가 둘 있다.

1. **1차 캐시 동일성 보장** — 하나의 영속성 컨텍스트 안에서 같은 ID 의 엔티티는 인스턴스가 딱 하나다.
   대부분의 비교는 `equals` 첫 줄 `if (this == o) return true;` 에서 즉시 끝난다.
2. **컬렉션 규모** — 한 회원이 쓴 글, 한 스터디의 참여자 같은 엔티티 컬렉션은 보통 수십~수백 개다. n 이 작다.

즉 **성능을 내주고 정확성을 산 거래**다. 정확성이 깨지면 데이터를 잃지만, 성능은 n 이 작으면 감당된다.

Lombok 을 유지해야 한다면 `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` + `id` 에만 `@EqualsAndHashCode.Include`.

> [!WARNING]
> **오답 코너**
> - **"엔티티에 Lombok `@EqualsAndHashCode` 를 붙이면 편하다"** — 전체 필드가 들어가 세 지뢰를 동시에 밟는다.
> - **"`getClass()` 프록시 문제의 해법은 `getId()` 로 비교하는 것"** — **`instanceof`** 다.
>   문제는 비교 대상이 아니라 그 앞의 **타입 체크 관문**이고, `getClass()` 에서 이미 `return false` 로 빠져나간다.
> - **"`@Id` 를 원시타입 `long` 으로 하면 안전하다"** — "값 없음"을 표현하지 못해 `isNew()` 판정이 무너진다
>   ([[wrapper-cache-autoboxing]]).
> - **"`equals` 는 예외를 던지지 않는 안전한 메서드"** — 지연 연관이 들어가면 컬렉션 연산 어디서든 터진다.

## 복습 체크

- [ ] 전체 필드 `@EqualsAndHashCode` 가 밟는 지뢰 세 개를 말할 수 있는가?
- [ ] 무한 재귀 호출 순서를 ①~④로 따라갈 수 있는가?
- [ ] `LazyInitializationException` 이 왜 `HashSet.add` 에서 터지는가?
- [ ] `getClass()` 와 `instanceof` 의 차이와 그 대가는?
- [ ] `hashCode` 에 넣을 수 있는 필드가 하나도 없는 이유를 후보별로 설명할 수 있는가?
- [ ] 상수 `hashCode` 의 성능 저하가 감당 가능한 두 가지 이유는?

## 관련

[[equals-hashcode-contract]] · [[wrapper-cache-autoboxing]] · [[lazy-loading-proxy]] · [[n-plus-1-problem]] · [[jvm-stack-and-heap]] · [[dirty-checking]]
