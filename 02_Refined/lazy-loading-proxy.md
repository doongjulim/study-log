---
type: refined
slug: lazy-loading-proxy
tags: [지연로딩, lazy-loading, LAZY, EAGER, 즉시로딩, 왜LAZY, LAZY기본값, 프록시, proxy, 프록시객체, HibernateProxy, 프록시구조, target, 위임, delegation, 상속, inheritance, 자식클래스, LazyInitializer, 프록시초기화, 초기화시점, LazyInitializationException, could-not-initialize-proxy, no-Session, Hibernate-initialize, 영속성컨텍스트, 준영속, detached, 필드직접접근, field-access, getter, getId, JPA, Hibernate, equals, hashCode, getClass, instanceof, 컬렉션연산, fetch-join, 면접답변]
topic: 지연 로딩 프록시의 정체(상속)·내부 구조(target 위임)·초기화 시점, LazyInitializationException 의 진짜 원인, 그리고 연관관계 기본값을 LAZY 로 두는 이유
summary: LAZY 연관은 원본을 상속한 프록시로 채워지고, ID 외의 값을 처음 읽을 때 영속성 컨텍스트에 초기화를 위임해 target 에 진짜 엔티티를 연결한 뒤 이후 호출을 target 에 넘긴다. 프록시 자신의 필드는 끝까지 비어 있고 가로챌 수 있는 건 메서드뿐이다. 컨텍스트가 닫힌 뒤 초기화하면 LazyInitializationException. LAZY 는 N+1 을 미룰 뿐이지만 fetch join 으로 쿼리마다 켤 수 있고 EAGER 는 끌 수 없어서 기본값으로 둔다.
contributors: [dongju]
source_refs:
  - https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html
updated: 2026-10-14
---

# 지연 로딩과 프록시

## 개념 정의

`@ManyToOne(fetch = FetchType.LAZY)` 같은 지연 연관은 조회 직후 **진짜 엔티티가 아니라 프록시 객체**로 채워진다.
프록시는 Hibernate 가 런타임에 만든 **원본 클래스의 자식 클래스**(`Member$HibernateProxy$...`)라서
`private Member member` 필드에 그대로 대입된다. 처음 아는 것은 **식별자(ID)뿐** — 부모 테이블의 FK(`post.member_id`)에서 얻은 값이다.

`EAGER` 는 조회 시점에 연관까지 즉시 채우지만, 쿼리 시점을 개발자가 통제할 수 없어
예측 불가능한 조인·N+1 을 유발하므로 **`@ManyToOne`/`@OneToOne` 도 명시적으로 `LAZY` 로 두는 것이 기본 권장**이다
(이 둘의 JPA 기본값이 `EAGER` 라는 점이 함정). 이유는 아래 "왜 기본값을 LAZY 로 두나" 참고.

## 작동 원리

### 초기화 시점

```java
postRepository.findAll();   // ① 쿼리 O
post.getTitle();            // ② 쿼리 X (Post 는 이미 조회됨)
post.getMember();           // ③ 쿼리 X (프록시를 꺼내기만 함)
    .getNickname();         // ④ 쿼리 O (ID 외의 값을 처음 읽음 → 초기화)
post.getMember().getId();   // 쿼리 X — 프록시가 이미 아는 값
```

### 프록시 초기화 흐름

1. `post.getMember()` — 아직 프록시. 쿼리 안 나감.
2. `post.getMember().getNickname()` — **실제 값이 필요해진 순간**
3. 프록시가 자기를 만들어준 **영속성 컨텍스트(Session)에 "채워달라"고 위임**
4. 컨텍스트가 `SELECT` 를 실행해 진짜 엔티티를 만들고 프록시의 **target** 에 연결
5. 이후 호출은 채워진 target 으로 위임

핵심은 **프록시가 스스로 쿼리를 날리지 않는다**는 것이다. 반드시 자기를 만든 Session 에 부탁해야 한다.

### 내부 구조 — target 참조와 위임

```
 post.member
     │
     ▼
┌───────────────────────┐            ┌─────────────────────┐
│ Member$HibernateProxy │   target   │ Member (진짜 엔티티)  │
│  id 필드   = null      │ ─────────▶ │  id = 1             │
│  name 필드 = null      │            │  nickname = "동주"   │
└───────────────────────┘            └─────────────────────┘
       ▲ getNickname() 호출              ▲ 초기화 시점에 DB 조회해서 생성
```

- 프록시는 진짜 엔티티에 대한 **참조**를 target 으로 가진다. 처음엔 `target == null`.
- 호출이 오면 **자기 필드가 아니라 target 에게 넘겨서(위임)** 결과를 돌려준다.
- 그래서 프록시 **자신의 필드는 끝까지 null** 이다 — 리플렉션으로 프록시의 필드를 직접 보면 초기화 후에도 null.
- 프록시가 "아는" id 는 자기 `id` 필드가 아니라 내부 초기화기(`LazyInitializer`)에 있고, `getId()` 를 **가로채서** 돌려준다.

### 프록시가 가로챌 수 있는 것은 메서드뿐이다

상속으로 자식 클래스가 재정의(override)할 수 있는 건 **메서드**뿐, **필드 접근은 가로챌 수 없다**.

```java
other.getId()   // 메서드 → 프록시가 가로챔 → 1
other.id        // 필드   → 가로챌 수 없음 → 프록시 자신의 필드 → null
```

엔티티 코드(특히 `equals`)에서 **다른 인스턴스의 필드를 직접 읽으면** 상대가 프록시일 때 조용히 null 을 읽는다 ([[jpa-entity-equality]]).

### `LazyInitializationException`

```
org.hibernate.LazyInitializationException:
    could not initialize proxy [Member#1] - no Session
```

> 프록시가 값을 채우려고 **Session 에 부탁했는데, 그 Session 이 이미 닫혀 있어서** 발생한다.
> **DB 도 커넥션도 멀쩡하다. 없는 것은 오직 "부탁할 대상"이다.**
> 이름 그대로 **Lazy** 프록시의 **Initialization** 실패다.

이 예외가 나오는 대표 상황:

- `spring.jpa.open-in-view=false` 인데 컨트롤러·뷰에서 프록시를 건드림 — OSIV off 면 영속성 컨텍스트는
  **서비스의 `@Transactional` 종료와 함께** 닫히고, 템플릿 렌더링은 그 **후**에 일어난다 ([[open-in-view-osiv]])
- `@Transactional` 밖에서 조회 결과를 받아 나중에 탐색
- 준영속(detached) 엔티티를 다른 계층으로 넘긴 뒤 탐색

## 트레이드오프 / 한계

**처방 3계열**

| 방식 | 내용 | 주의 |
|---|---|---|
| 트랜잭션 안에서 미리 채우기 | `join fetch`, `@EntityGraph` | 컬렉션 fetch join 은 페이징 불가 ([[n-plus-1-problem]]) |
| 강제 초기화 | `Hibernate.initialize(post.getMember())` | 개수만큼 쿼리가 나가 N+1 위험 |
| 엔티티를 안 넘기기 | 트랜잭션 안에서 DTO 로 조회·변환해 반환 | 계층 경계가 가장 깔끔 ([[jpa-dto-projection]]) |

**OSIV 를 켜서 회피하는 것은 처방이 아니다.** 예외는 사라지지만 쿼리 발생 지점이 뷰 계층으로 흩어지고
커넥션 점유가 길어진다. 예외는 "데이터를 트랜잭션 안에서 확정 짓지 않았다"는 설계 신호로 읽는 편이 낫다.

### 왜 기본값을 LAZY 로 두나 — 켜기는 되고, 끄기는 안 된다

LAZY 도 N+1 을 **없애지는 않는다.** 루프에서 연관 값을 읽으면 프록시가 하나씩 초기화되어 1 + N 이 그대로 나간다.
그런데도 LAZY 를 기본으로 두는 이유는 **비대칭** 때문이다.

| | 특정 쿼리에서만 바꾸기 |
|---|---|
| LAZY 로 둔 연관 | fetch join / `@EntityGraph` 로 **"이번만 함께 가져와"** — 가능 |
| EAGER 로 둔 연관 | **"이번엔 가져오지 마"** — 방법이 없다 |

게다가 EAGER 는 JPQL 조회에서 조인이 아니라 N+1 로 가져온다(JPQL 은 fetch 설정과 무관하게 번역된 뒤 추가 조회).

> **면접 답변** — "EAGER 는 JPQL 로 조회할 때 쿼리가 그대로 번역된 뒤 연관 엔티티를 추가로 조회해서 N+1 이 생기고,
> 쓰지 않는 데이터까지 항상 가져옵니다. 게다가 특정 쿼리에서만 끄는 방법이 없습니다.
> LAZY 는 프록시로 조회를 미뤄 두었다가 필요한 화면에서만 fetch join 이나 @EntityGraph 로 한 번에 가져올 수 있습니다.
> LAZY 자체가 N+1 을 없애 주지는 않지만, 조회 범위를 쿼리마다 제가 통제할 수 있기 때문에 기본값을 LAZY 로 두었습니다."

> [!WARNING]
> **오답 코너**
> - **"영속성 컨텍스트가 죽은 상태에서 DB 를 건드려서 터진다"** — DB 를 건드리려 한 게 문제가 아니라
>   **건드려 달라고 부탁할 대상(Session)이 없는 것**이 원인이다. 커넥션·DB 는 정상이다.
> - **"LAZY 면 쿼리가 안 나간다" / "LAZY 로 N+1 을 해결했다"** — 미루는 것이지 없애는 것이 아니다. 값을 건드리는 순간 나간다.
>   면접에서 "LAZY 로 N+1 해결"이라고 말하면 "작성자 이름 출력하면요?" 반격이 들어온다.
> - **"`@ManyToOne` 기본값은 LAZY"** — `@ManyToOne`/`@OneToOne` 의 JPA 기본값은 **`EAGER`** 다.
>   `@OneToMany`/`@ManyToMany` 만 `LAZY` 가 기본.
> - **"`getMember()` 를 호출하는 순간 DB 조회"** — 프록시를 **꺼내기만** 할 땐 쿼리가 없다. ID 외의 값을 읽을 때 초기화.
> - **"프록시와 원본의 관계는 조인"** — 조인은 DB 테이블 개념. 자바 타입 관계는 **상속**이다.
> - **"프록시가 target 정보를 위임받아 들고 있다"** — 방향 반대. 프록시는 target **참조**만 들고, 호출을 target 에 **넘긴다**.
> - **"프록시는 id 를 알고 있으니 `other.id` 도 값이 나온다"** — 필드 접근은 가로챌 수 없어 프록시 자신의 필드(null)를 읽는다.
>   id 를 아는 건 `getId()` 를 가로채기 때문이다.

### equals / hashCode 안의 지연 연관 — 예외가 엉뚱한 곳에서 터진다

이 예외는 보통 "컨트롤러·뷰에서 지연 연관을 건드릴 때 나는 것"으로 알려져 있지만,
`equals`/`hashCode` 에 지연 연관 필드가 포함되면 **발생 지점이 완전히 달라진다.**

```java
@Entity @EqualsAndHashCode           // 모든 필드 포함 → writer 프록시도 비교 대상
class Notice { @Id Long id; String title; @ManyToOne(fetch = LAZY) Member writer; }
```

`set.add()`, `list.contains()`, `assertEquals()` 같은 **평범한 컬렉션 연산이 예외 발생 지점**이 된다.
`equals` 는 아무도 터질 거라고 예상하지 않는 메서드이고, 스택 트레이스도 `HashSet.add` 로 찍혀
원인을 가린다. 그래서 진단이 훨씬 어렵다 ([[jpa-entity-equality]]).

### getClass() 는 프록시를 다른 클래스로 판정한다

프록시는 원본 클래스를 **상속한** 껍데기(`Member$HibernateProxy$abc123`)이므로,

| | 원본 vs 프록시 |
|---|---|
| `getClass() != o.getClass()` | **다르다고 판정** → 식별자 비교까지 가지도 못함 |
| `o instanceof Member` | 하위 타입이므로 **통과** |

엔티티 `equals` 에서 `instanceof` 를 쓰는 이유가 이것이다. 대가로 상속 계층에서 대칭성이 깨질 여지를 받아들인다.
Hibernate 6 라면 `Hibernate.getClass(o)` 로 프록시를 벗겨낼 수도 있다.

## 복습 체크

- [ ] LAZY 연관 필드에 처음 담기는 것이 무엇인지, 원본과 어떤 관계(상속)이고 무엇을 알고 있는지 말할 수 있는가?
- [ ] `findAll` → `getTitle` → `getMember` → `getNickname` → `getId` 중 쿼리가 나가는 줄을 고를 수 있는가?
- [ ] 프록시가 실제 값을 채우는 5단계를 순서대로 설명할 수 있는가?
- [ ] target 과 위임으로 프록시 구조를 그리고, 초기화 후에도 프록시 자신의 필드가 null 인 이유를 말할 수 있는가?
- [ ] 프록시가 `getId()` 는 가로채지만 `other.id` 는 못 가로채는 이유는?
- [ ] `LazyInitializationException` 의 원인을 "DB" 가 아니라 "Session" 으로 정확히 지목할 수 있는가?
- [ ] OSIV off 에서 영속성 컨텍스트가 닫히는 시점과 렌더링 시점의 선후를 말할 수 있는가?
- [ ] 이 예외가 나오는 대표 상황 세 가지를 댈 수 있는가?
- [ ] 연관관계 애노테이션별 fetch 기본값을 정확히 말할 수 있는가?
- [ ] LAZY 가 N+1 을 해결하지 않는데도 기본값으로 두는 이유(비대칭)를 30초로 답할 수 있는가?
- [ ] `equals` 안에 지연 연관이 들어가면 예외 발생 지점이 어떻게 달라지는가?
- [ ] `getClass()` 가 프록시를 어떻게 판정하며, 대안은 무엇인가?

## 관련

[[open-in-view-osiv]] · [[n-plus-1-problem]] · [[jpa-dto-projection]] · [[dirty-checking]] · [[jpa-entity-equality]]
