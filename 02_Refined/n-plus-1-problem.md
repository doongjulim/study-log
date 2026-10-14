---
type: refined
slug: n-plus-1-problem
tags: [N+1, N+1문제, n-plus-one, 1+N, fetch-join, join-fetch, left-join-fetch, 페치조인, EntityGraph, 엔티티그래프, default_batch_fetch_size, batch-size, IN절배치, 지연로딩쿼리폭발, EAGER, LAZY, 즉시로딩, JPQL, em.find, findAll, 1차캐시, MyBatis, 접근패턴, access-pattern, 카테시안곱, 행뻥튀기, row-explosion, 컬렉션조인, 1:N, N:1, 컬렉션페이징, 메모리페이징, firstResult-maxResults, HHH000104, HHH90003004, MultipleBagFetchException, DTO조회, projection, select-new, size(), 서브쿼리, 반정규화, denormalization, 카운트N+1, JPA, Hibernate, equals, hashCode, EqualsAndHashCode, 컬렉션연산, 숨은N+1]
topic: 목록 쿼리 1번 뒤 행마다 연관 데이터를 따로 조회해 쿼리가 1+N 번 나가는 문제 — EAGER/LAZY 모두에서 생기는 이유와 fetch join·@EntityGraph·batch size·DTO 조회·반정규화 처방 비교
summary: 목록 쿼리 1번 + 결과 행마다 연관 데이터 조회 N번. 원인은 행 단위로 따로 조회하는 접근 패턴이라 MyBatis 에서도 생기고, JPA 에서는 EAGER(JPQL 은 그대로 번역된 뒤 추가 조회)와 LAZY(초기화 시점으로 미뤄질 뿐) 모두에서 생긴다. fetch join·@EntityGraph(1번)·batch size(1+1번)·DTO 조회(1번)로 풀고, 1:N 컬렉션 조인은 행이 불어나 페이징을 깨뜨리므로 목록에서는 피한다.
contributors: [dongju]
source_refs:
  - https://docs.spring.io/spring-data/jpa/reference/jpa/entity-persistence.html
  - https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html
updated: 2026-10-14
---

# N+1 문제와 fetch join

## 개념 정의

> **N+1 문제**는 목록을 조회하는 쿼리 **1번**을 실행한 뒤, 결과 행마다 연관 데이터를 FK 로 하나씩 조회하는 쿼리가
> **N번** 더 나가는 문제다. 연관 데이터를 첫 쿼리에서 함께 가져오지 않고 **행 단위로 따로** 가져오기 때문에 생긴다.

```java
List<Post> posts = postRepository.findAll();          // ① SELECT * FROM post
for (Post p : posts) {
    System.out.println(p.getMember().getNickname());  // ②~ 행마다 SELECT * FROM member WHERE id=?
}
```

글이 100건이면 **1 + 100 = 101번**. 목록 화면에서 조용히 성능을 갉아먹는 대표적 패턴이며,
개발 DB 에서는 데이터가 적어 잘 드러나지 않다가 운영에서 터진다.

- **N 은 정확히는 "서로 다른 연관 엔티티 수"** — 이미 조회한 엔티티는 1차 캐시에서 꺼낸다.
  글 20개에 작성자가 3명이면 추가 쿼리는 3번 ([[persistence-context]]).
- 정의에 EAGER/LAZY 를 넣지 않는다. **두 설정 모두에서 생기므로** fetch 설정은 원인이 아니다.

### JPA 만의 문제가 아니다

```java
List<Post> posts = postMapper.findAll();                       // select * from post
for (Post post : posts) {
    Member m = memberMapper.findById(post.getMemberId());     // select * from member where id = ?
}
```

프록시도 EAGER 도 없는 MyBatis 코드인데 똑같이 1 + N. N+1 은 **데이터 접근 패턴**의 문제이고,
JPA 는 이 루프가 `getNickname()` 한 줄 뒤에 **숨어서 안 보일** 뿐이다.

### EAGER 도 N+1 이다 — JPQL 은 fetch 설정을 모른다

| 호출 | 실제 쿼리 |
|---|---|
| `em.find(Post.class, 1L)` | `post` + `member` **조인 1번** |
| `postRepository.findAll()` (JPQL) | **N+1** |

```
① JPQL "select p from Post p" → fetch 설정과 무관하게 글자 그대로 SQL 로 번역 (member 조인 없음)
② 결과 Post 들이 영속성 컨텍스트에 올라옴
③ 매핑 정보를 보니 member 가 EAGER → 즉시 채우려고 member 마다 추가 SELECT
```

> **JPQL 은 그대로 번역되고, fetch 설정은 조회 이후 시점에 반영된다.**

### LAZY 는 N+1 을 없애지 않고 미룰 뿐이다

LAZY 면 `findAll()` 직후엔 쿼리가 1번이지만, 루프에서 `getMember().getNickname()` 을 읽는 순간
프록시가 하나씩 초기화된다. 작성자 20명이면 **1 + 20 = 21번** — 결과는 EAGER 와 같다 ([[lazy-loading-proxy]]).
트랜잭션 안에서 for 문으로 프록시를 초기화하는 것도 예외를 피할 뿐 N+1 은 그대로다.

## 작동 원리 — 처방

### 한눈에 비교

| 방법 | 쿼리 수 | 결과 | 분류 |
|---|---|---|---|
| ① `join fetch` | 1 | 엔티티 | 쿼리 기법 (JPQL 에 직접) |
| ② `@EntityGraph` | 1 | 엔티티 | 쿼리 기법 (어노테이션으로 fetch join) |
| ③ `default_batch_fetch_size` | 1+1 | 엔티티 | 프록시 초기화 방식 변경 |
| ④ DTO 직접 조회 (`select new`) | 1 | DTO | 쿼리 기법 |
| ⑤ 반정규화 | 1 | 엔티티 | **설계 선택** — 연관 조회 자체를 없앰 |

### ① fetch join (JPQL)

```java
@Query("select p from Post p left join fetch p.member")
List<Post> findAllWithMember();
```

**한 방의 조인 쿼리로 연관 엔티티까지 영속성 컨텍스트에 채워온다.** 이후 프록시 초기화가 아예 필요 없다.
fetch 설정으로는 조인 SQL 을 만들 수 없으므로(JPQL 은 그대로 번역) 조인은 **JPQL 자체**에 써야 한다.

일반 `join` 과의 차이 — 일반 join 은 조인만 하고 **SELECT 절에는 루트(Post) 컬럼만** 올린다.

```sql
-- select p from Post p left join p.member
select p.id, p.title, p.member_id      -- member 컬럼 없음 → p.member 는 여전히 프록시
from post p left join member m on p.member_id = m.id
```

`left`/`inner` 는 **어떤 행을 가져오느냐**(연관이 null 인 행 포함 여부)의 문제이고, **프록시가 남느냐**와는 별개다.

### ② `@EntityGraph` (Spring Data JPA)

```java
@EntityGraph(attributePaths = {"member"})
List<Post> findAll();
```

하는 일은 fetch join 과 같다. JPQL 직접 작성이냐 애노테이션 선언이냐의 차이.

### ③ `default_batch_fetch_size` — 조회 쿼리는 그대로, 초기화를 묶는다

```yaml
spring.jpa.properties.hibernate.default_batch_fetch_size: 100
```

```
① findAll()          → select * from post                       (1번, member 는 프록시)
② 첫 getNickname()   → 설정 없으면: where id = ?                (프록시 하나만 초기화)
                        설정 있으면: where id in (?, ?, ..., ?)   (미초기화 프록시를 모아 한 번에)
```

N+1 이 **1 + (N/배치크기)** 로 줄어든다. 초기화되는 건 진짜 엔티티이므로 결과는 **엔티티**.
컬렉션이 여러 개이거나 컬렉션 + 페이징처럼 fetch join 으로 다 풀 수 없는 상황의 표준 처방.

### ④ DTO 직접 조회

애초에 엔티티를 안 가져오고 필요한 컬럼만 `select new` 로 뽑는다. 영속성 컨텍스트·프록시 자체가 개입하지 않는다
([[jpa-dto-projection]]).

### ⑤ 반정규화

연관 테이블의 값을 부모 테이블에 **복사**해 두어 연관 조회 자체를 없앤다(예: 글마다 `count(*)` 대신 `like_count` 컬럼).
쿼리 기법이 아니라 **설계 선택**이며, 원본과 복사본을 직접 맞춰야 하는 대가가 따른다 ([[denormalized-counter-atomic-update]]).

## 트레이드오프 / 한계

**컬렉션(1:N) fetch join 의 두 가지 벽**

1. **페이징 불가 — 행 뻥튀기** — 글 2개, 파일이 3개·2개일 때 `files` 를 조인하면:

   ```
   post.id | title | file.id
   --------+-------+--------
      1    | 글A   |  10
      1    | 글A   |  11      ← 부모 행이 자식 수만큼 불어남
      1    | 글A   |  12
      2    | 글B   |  20
      2    | 글B   |  21
   ```

   여기서 `limit 2` 면 글A 행 2개만 잘린다. 그래서 Hibernate 는 경고를 남기고 **전체를 읽어 메모리에서 페이징**한다
   (Hibernate 5: `HHH000104`, Hibernate 6: `HHH90003004` — `firstResult/maxResults specified with collection fetch; applying in memory`).
   사실상 OOM 지뢰. ([[db-pagination]])
   - **N:1 조인은 안전하다.** 작성자는 글마다 하나라 행이 불어나지 않는다. 불어나는 건 **1:N 컬렉션**뿐.
   - **페이징 없는 단건 조회**라면 컬렉션 fetch join 도 안전하다 — 불어난 행이 부모 하나로 합쳐지면 끝.
2. **컬렉션 둘 이상 동시 fetch join 불가** — 카테시안 곱이 발생하며 `MultipleBagFetchException`.

**선택 기준**

| 상황 | 처방 |
|---|---|
| `@ManyToOne` 단일 연관 | fetch join / `@EntityGraph` |
| 단건 상세 + 컬렉션 | fetch join / `@EntityGraph` (페이징 없으므로 안전) |
| 목록 + 컬렉션 **엔티티**가 필요 | 부모만 페이징 + `default_batch_fetch_size` |
| 목록 + 컬렉션은 **개수만** 필요 | DTO 조회 + `size(p.files)` (count 서브쿼리, 행이 안 불어남) |
| 화면 전용 데이터, 수정 안 함 | DTO 직접 조회 |
| 집계값을 목록마다 반복 계산 | 반정규화 카운터 (정합성 책임 감수) |

### 사례 — OSIV off 인 게시판에서의 선택

OSIV 를 끄면 화면에 쓸 데이터를 **트랜잭션 안에서 다 채워서** 내보내야 한다 ([[open-in-view-osiv]]).
N+1 해결과 OSIV 대응을 한 번에 하는 조합:

```java
// 목록: 페이징 O → 컬렉션을 조인하지 않고 개수만 서브쿼리로
@Query("select new com.example.board.post.dto.PostSummary(" +
       "p.id, p.title, p.author.nickname, p.category, p.createdAt, size(p.files), p.viewCount, p.likeCount) " +
       "from Post p where ...")
Page<PostSummary> findSummaries(..., Pageable pageable);

// 상세: 페이징 없는 단건 → 컬렉션까지 함께
@EntityGraph(attributePaths = {"author", "files"})
Optional<Post> findById(Long id);
```

- `p.author.nickname` 은 N:1 암묵적 조인이라 페이징에 안전하다.
- `size(p.files)` 는 `(select count(*) from attached_file where post_id = p.id)` 같은 서브쿼리 → 글 하나당 숫자 하나.

> [!WARNING]
> **오답 코너**
> - **"트랜잭션 안에서 조회하면 N+1 이 해결된다"** — 해결되는 것은 `LazyInitializationException` 뿐이다.
>   쿼리 횟수는 그대로 101번.
> - **"LAZY 로 바꾸면 N+1 이 해결된다"** — 시점을 미룰 뿐이다. 연관 값을 루프에서 읽으면 1 + N 그대로.
> - **"`join` 과 `join fetch` 는 같다" / "`left join` 이면 해결된다"** — 일반 join 은 연관을 SELECT 절에 넣지 않아
>   프록시가 남는다. left/inner 는 행 포함 여부의 문제다.
> - **"EAGER 로 바꾸면 N+1 이 사라진다"** — JPQL 은 fetch 설정과 무관하게 번역된 뒤 EAGER 연관을 추가 조회하므로
>   N+1 이 그대로 나고, 쓰지 않는 화면에서도 항상 나간다.
> - **"JPQL 을 SQL 로 바꿀 때 EAGER 설정을 참고한다"** — 참고한다면 조인 1번이었어야 한다. 그대로 번역되고 조회 **이후**에 반영된다.
> - **"조인해서 같이 가져와야 해서 N+1 이 생긴다"** — 반대. 같이 가져오지 **않고** 행마다 따로 가져와서 생긴다.
> - **정의를 "ID 외의 값을 읽을 때 한 번 더 조회"로 설명** — LAZY 프록시에서만 맞는 설명. EAGER·MyBatis 를 포괄하지 못한다.
> - **"batch size 는 쿼리 1번, 결과는 DTO"** — **1+1번, 엔티티**. 조회 쿼리가 아니라 초기화 방식을 바꾼다.
> - **"조인된 엔티티 개수만큼 목록 행이 늘어난다"** — **1:N 컬렉션**만 불어난다. N:1 조인은 페이징에 안전.
> - **"컬렉션도 fetch join 하고 페이징하면 된다"** — 메모리 페이징으로 조용히 전환된다. 로그의 `HHH90003004`(구 `HHH000104`)를 볼 것.
> - **용어** — `left fetch join` 이 아니라 **`left join fetch`**. FK 를 "시퀀스값"이라 부르지 말 것(시퀀스는 PK 생성기).

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

### 면접 답변

> "N+1 은 목록을 조회하는 쿼리 1번 뒤에, 결과 행마다 연관 데이터를 따로 조회하는 쿼리가 N번 더 나가는 문제입니다.
> 연관 데이터를 행 단위로 따로 가져오는 패턴이 원인이라 JPA 뿐 아니라 MyBatis 에서도 생깁니다.
> 해결책은 fetch join 과 @EntityGraph(쿼리 1번, 엔티티), batch fetch size(in 쿼리로 1+1번), DTO 조회(1번, 필요한 값만)가 있습니다.
> 목록처럼 페이징이 있는 곳에서 1:N 컬렉션을 조인하면 행이 불어나 페이지가 깨지므로, 컬렉션은 개수만 DTO 로 받거나 batch size 로 채우고,
> 페이징 없는 단건 상세에서만 컬렉션을 함께 가져옵니다."

## 복습 체크

- [ ] N+1 을 EAGER/LAZY 라는 단어 없이 한 문장으로 정의하고, 1 과 N 이 각각 어떤 쿼리인지 말할 수 있는가?
- [ ] N+1 이 MyBatis 에서도 생기는 이유를 설명할 수 있는가?
- [ ] 글 100건 목록에서 쿼리가 몇 번 나가는지, 작성자가 3명뿐이면 몇 번인지 설명할 수 있는가?
- [ ] `em.find` 는 조인 1번인데 `findAll()` 은 왜 N+1 인가?
- [ ] LAZY 로 바꾼 뒤 루프에서 연관 값을 읽으면 쿼리가 몇 번인가?
- [ ] `join` 과 `join fetch` 의 차이를 SELECT 절 기준으로 한 문장으로 말할 수 있는가?
- [ ] 처방 다섯 가지의 쿼리 수와 결과 타입(엔티티/DTO)을 말할 수 있는가?
- [ ] `default_batch_fetch_size` 가 쿼리 횟수를 어떤 식으로 줄이는가? 조회 쿼리 자체를 바꾸는가?
- [ ] 1:N 컬렉션 조인이 페이징을 깨뜨리는 이유를 행 그림으로 설명할 수 있는가? N:1 은 왜 안전한가?
- [ ] 컬렉션 fetch join 에 페이징을 걸면 무슨 일이 벌어지는가? 로그에 뭐가 찍히는가?
- [ ] 단건 상세에서는 컬렉션 fetch join 이 왜 괜찮은가?
- [ ] 목록에 컬렉션 **개수**만 필요할 때의 처방은?
- [ ] 컬렉션 두 개를 동시에 fetch join 하면 어떤 예외가 나는가?
- [ ] `equals`/`hashCode` 가 N+1 을 유발하는 경로와, 이 경우 `join fetch` 가 처방이 되지 못하는 이유는?

## 관련

[[lazy-loading-proxy]] · [[jpa-dto-projection]] · [[denormalized-counter-atomic-update]] · [[open-in-view-osiv]] · [[persistence-context]] · [[db-pagination]] · [[keyset-pagination]] · [[jpa-entity-equality]] · [[equals-hashcode-contract]]
