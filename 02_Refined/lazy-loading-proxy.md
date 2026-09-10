---
type: refined
slug: lazy-loading-proxy
tags: [지연로딩, lazy-loading, LAZY, EAGER, 즉시로딩, 프록시, proxy, 프록시객체, LazyInitializationException, could-not-initialize-proxy, no-Session, 프록시초기화, Hibernate-initialize, 영속성컨텍스트, 준영속, detached, JPA, Hibernate, equals, hashCode, getClass, instanceof, HibernateProxy, 컬렉션연산]
topic: 지연 로딩 프록시가 실제 값을 채우는 메커니즘과 LazyInitializationException 의 진짜 원인
summary: LAZY 연관은 껍데기 프록시로 채워지고, 실제 값 접근 시 영속성 컨텍스트에 쿼리를 위임해 초기화된다. 컨텍스트가 닫힌 뒤 프록시를 건드리면 "부탁할 대상이 없어" LazyInitializationException 이 터진다.
contributors: [dongju]
source_refs:
  - https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html
updated: 2026-09-10
---

# 지연 로딩과 프록시

## 개념 정의

`@ManyToOne(fetch = FetchType.LAZY)` 같은 지연 연관은 조회 직후 **진짜 엔티티가 아니라 프록시 객체**로 채워진다.
프록시는 원본 클래스를 상속한 **껍데기**로, 식별자(ID)만 알고 나머지 필드는 비어 있다.

`EAGER` 는 조회 시점에 연관까지 즉시 채우지만, 쿼리 시점을 개발자가 통제할 수 없어
예측 불가능한 조인·N+1 을 유발하므로 **`@ManyToOne`/`@OneToOne` 도 명시적으로 `LAZY` 로 두는 것이 기본 권장**이다
(이 둘의 JPA 기본값이 `EAGER` 라는 점이 함정).

## 작동 원리

프록시 초기화 흐름:

1. `post.getMember()` — 아직 프록시. 쿼리 안 나감.
2. `post.getMember().getNickname()` — **실제 값이 필요해진 순간**
3. 프록시가 자기를 만들어준 **영속성 컨텍스트(Session)에 "채워달라"고 위임**
4. 컨텍스트가 `SELECT` 를 실행하고 프록시 내부의 target 을 채움
5. 이후 호출은 채워진 target 으로 위임

핵심은 **프록시가 스스로 쿼리를 날리지 않는다**는 것이다. 반드시 자기를 만든 Session 에 부탁해야 한다.

### `LazyInitializationException`

```
org.hibernate.LazyInitializationException:
    could not initialize proxy [Member#1] - no Session
```

> 프록시가 값을 채우려고 **Session 에 부탁했는데, 그 Session 이 이미 닫혀 있어서** 발생한다.
> **DB 도 커넥션도 멀쩡하다. 없는 것은 오직 "부탁할 대상"이다.**

이 예외가 나오는 대표 상황:

- `spring.jpa.open-in-view=false` 인데 컨트롤러·뷰에서 프록시를 건드림 ([[open-in-view-osiv]])
- `@Transactional` 밖에서 조회 결과를 받아 나중에 탐색
- 준영속(detached) 엔티티를 다른 계층으로 넘긴 뒤 탐색

## 트레이드오프 / 한계

**처방 3계열**

| 방식 | 내용 | 주의 |
|---|---|---|
| 트랜잭션 안에서 미리 채우기 | `join fetch`, `@EntityGraph` | 컬렉션 fetch join 은 페이징 불가 ([[n-plus-1-problem]]) |
| 강제 초기화 | `Hibernate.initialize(post.getMember())` | 개수만큼 쿼리가 나가 N+1 위험 |
| 엔티티를 안 넘기기 | 트랜잭션 안에서 DTO 로 변환해 반환 | 계층 경계가 가장 깔끔 |

**OSIV 를 켜서 회피하는 것은 처방이 아니다.** 예외는 사라지지만 쿼리 발생 지점이 뷰 계층으로 흩어지고
커넥션 점유가 길어진다. 예외는 "데이터를 트랜잭션 안에서 확정 짓지 않았다"는 설계 신호로 읽는 편이 낫다.

> [!WARNING]
> **오답 코너**
> - **"영속성 컨텍스트가 죽은 상태에서 DB 를 건드려서 터진다"** — DB 를 건드리려 한 게 문제가 아니라
>   **건드려 달라고 부탁할 대상(Session)이 없는 것**이 원인이다. 커넥션·DB 는 정상이다.
> - **"LAZY 면 쿼리가 안 나간다"** — 미루는 것이지 없애는 것이 아니다. 값을 건드리는 순간 나간다.
> - **"`@ManyToOne` 기본값은 LAZY"** — `@ManyToOne`/`@OneToOne` 의 JPA 기본값은 **`EAGER`** 다.
>   `@OneToMany`/`@ManyToMany` 만 `LAZY` 가 기본.

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

- [ ] LAZY 연관 필드에 처음 담기는 것이 무엇인지, 그 안에 무엇이 들어있는지 말할 수 있는가?
- [ ] 프록시가 실제 값을 채우는 5단계를 순서대로 설명할 수 있는가?
- [ ] `LazyInitializationException` 의 원인을 "DB" 가 아니라 "Session" 으로 정확히 지목할 수 있는가?
- [ ] 이 예외가 나오는 대표 상황 세 가지를 댈 수 있는가?
- [ ] 연관관계 애노테이션별 fetch 기본값을 정확히 말할 수 있는가?
- [ ] `equals` 안에 지연 연관이 들어가면 예외 발생 지점이 어떻게 달라지는가?
- [ ] `getClass()` 가 프록시를 어떻게 판정하며, 대안은 무엇인가?

## 관련

[[open-in-view-osiv]] · [[n-plus-1-problem]] · [[dirty-checking]] · [[jpa-entity-equality]]
