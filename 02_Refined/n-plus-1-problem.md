---
type: refined
slug: n-plus-1-problem
tags: [N+1, N+1문제, n-plus-one, fetch-join, join-fetch, EntityGraph, 엔티티그래프, default_batch_fetch_size, batch-size, IN절배치, 지연로딩쿼리폭발, 카테시안곱, 컬렉션페이징, firstResult-maxResults, MultipleBagFetchException, DTO조회, projection, JPA, Hibernate, equals, hashCode, EqualsAndHashCode, 컬렉션연산, 숨은N+1]
topic: 연관 엔티티를 하나씩 초기화하다 쿼리가 1+N 번 나가는 문제와 fetch join / batch size / DTO 조회 처방
summary: 목록 조회 1번 + 각 행의 연관 초기화 N번으로 쿼리가 폭발하는 문제. fetch join·@EntityGraph 로 한 방에 채우거나, batch size 로 IN 절 묶음 조회를 하거나, 애초에 DTO 로 직접 조회한다.
contributors: [dongju]
source_refs:
  - https://docs.spring.io/spring-data/jpa/reference/jpa/entity-persistence.html
  - https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html
updated: 2026-09-10
---

# N+1 문제와 fetch join

## 개념 정의

목록을 조회한 뒤 각 행의 지연 연관을 하나씩 초기화하면서
**본 쿼리 1번 + 행 개수 N번**의 쿼리가 나가는 현상.

```java
List<Post> posts = postRepository.findAll();          // ① SELECT * FROM post
for (Post p : posts) {
    System.out.println(p.getMember().getNickname());  // ②~ 행마다 SELECT * FROM member WHERE id=?
}
```

글이 100건이면 **1 + 100 = 101번**. 목록 화면에서 조용히 성능을 갉아먹는 대표적 패턴이며,
개발 DB 에서는 데이터가 적어 잘 드러나지 않다가 운영에서 터진다.

트랜잭션 안에서 for 문으로 프록시를 초기화하는 것도 예외를 피할 뿐 N+1 은 그대로다
([[lazy-loading-proxy]]).

## 작동 원리 — 처방

### ① fetch join (JPQL)

```java
@Query("select p from Post p join fetch p.member")
List<Post> findAllWithMember();
```

**한 방의 조인 쿼리로 연관 엔티티까지 영속성 컨텍스트에 채워온다.** 이후 프록시 초기화가 아예 필요 없다.
일반 `join` 과의 차이: 일반 join 은 조인만 하고 SELECT 대상에 연관을 포함하지 않아 N+1 이 그대로 남는다.

### ② `@EntityGraph` (Spring Data JPA)

```java
@EntityGraph(attributePaths = {"member"})
List<Post> findAll();
```

하는 일은 fetch join 과 같다. JPQL 직접 작성이냐 애노테이션 선언이냐의 차이.

### ③ `default_batch_fetch_size` — 컬렉션에 대한 현실적 해법

```yaml
spring.jpa.properties.hibernate.default_batch_fetch_size: 100
```

지연 로딩이 발생할 때 **해당 대상들을 모아 `WHERE id IN (?, ?, ...)` 한 방으로** 가져온다.
N+1 이 **1 + (N/배치크기)** 로 줄어든다. 컬렉션이 여러 개일 때 fetch join 으로 다 풀 수 없는 상황의 표준 처방.

### ④ DTO 직접 조회

애초에 엔티티를 안 가져오고 필요한 컬럼만 projection 으로 뽑는다. 영속성 컨텍스트·프록시 자체가 개입하지 않는다.

## 트레이드오프 / 한계

**컬렉션 fetch join 의 두 가지 벽**

1. **페이징 불가** — `@OneToMany` 를 fetch join 하면 조인 결과 행 수가 부모 개수와 달라져
   DB 의 `LIMIT`/`OFFSET` 을 신뢰할 수 없다. Hibernate 는 경고를 남기고 **전체를 읽어 메모리에서 페이징**한다
   (`HHH000104: firstResult/maxResults specified with collection fetch; applying in memory`). 사실상 OOM 지뢰.
   → 이때가 `default_batch_fetch_size` 를 쓸 자리다. 페이징은 부모에만 걸고 컬렉션은 배치로 채운다.
   ([[db-pagination]])
2. **컬렉션 둘 이상 동시 fetch join 불가** — 카테시안 곱이 발생하며 `MultipleBagFetchException`.

**선택 기준**

| 상황 | 처방 |
|---|---|
| `@ManyToOne` 단일 연관 | fetch join / `@EntityGraph` |
| 컬렉션 + 페이징 | 부모만 페이징 + `default_batch_fetch_size` |
| 화면 전용 데이터, 수정 안 함 | DTO 직접 조회 |

> [!WARNING]
> **오답 코너**
> - **"트랜잭션 안에서 조회하면 N+1 이 해결된다"** — 해결되는 것은 `LazyInitializationException` 뿐이다.
>   쿼리 횟수는 그대로 101번.
> - **"`join` 과 `join fetch` 는 같다"** — 일반 join 은 연관을 SELECT 대상에 넣지 않아 N+1 이 남는다.
> - **"EAGER 로 바꾸면 N+1 이 사라진다"** — 오히려 악화된다. 쿼리 시점을 통제할 수 없어 예상 못 한 지점에서 터진다.
> - **"컬렉션도 fetch join 하고 페이징하면 된다"** — 메모리 페이징으로 조용히 전환된다. 로그의 `HHH000104` 를 볼 것.

### 코드에 쿼리가 보이지 않는 N+1 — equals / hashCode

`@EqualsAndHashCode`(전체 필드)가 붙은 엔티티는 `hashCode()` 계산에 지연 연관 필드가 포함된다.

```java
Set<Notice> set = new HashSet<>();
for (Notice n : notices) set.add(n);   // 1000개
```

`add()` → `hashCode()` 1000회 → 각각 프록시 초기화 → **`SELECT` 1000번.**
쿼리를 유발할 만한 코드가 한 줄도 보이지 않는데 쿼리가 쏟아진다.

**`join fetch` 로 아무리 최적화해도 소용이 없다.** 조회는 한 번에 끝났더라도
`equals`/`hashCode` 가 뒤에서 다시 프록시를 깨우기 때문이다.
연관 필드를 비교 대상에서 빼는 것이 근본 처방이다 ([[jpa-entity-equality]]).

> 참고로 이것은 **쿼리 횟수** 문제이며, 해시 충돌로 인한 O(n) 선형 탐색은 **비교 횟수** 문제다.
> 이름이 비슷해 헷갈리기 쉽지만 원인도 해법도 다르다 ([[equals-hashcode-contract]]).

## 복습 체크

- [ ] 글 100건 목록에서 쿼리가 몇 번 나가는지, 그 구성(1 + N)을 설명할 수 있는가?
- [ ] `join` 과 `join fetch` 의 차이를 한 문장으로 말할 수 있는가?
- [ ] `@EntityGraph` 와 fetch join 의 관계를 설명할 수 있는가?
- [ ] 컬렉션 fetch join 에 페이징을 걸면 무슨 일이 벌어지는가? 로그에 뭐가 찍히는가?
- [ ] `default_batch_fetch_size` 가 쿼리 횟수를 어떤 식으로 줄이는가?
- [ ] 컬렉션 두 개를 동시에 fetch join 하면 어떤 예외가 나는가?
- [ ] `equals`/`hashCode` 가 N+1 을 유발하는 경로를 설명할 수 있는가?
- [ ] 이 경우 `join fetch` 가 왜 처방이 되지 못하는가?

## 관련

[[lazy-loading-proxy]] · [[open-in-view-osiv]] · [[db-pagination]] · [[keyset-pagination]] · [[jpa-entity-equality]] · [[equals-hashcode-contract]]
