---
date: 2026-10-13
type: raw
author: dongju
tags: [지연 로딩, LAZY, 즉시 로딩, EAGER, 왜 LAZY, fetch join, 페치 조인, join fetch, left join fetch, 일반 조인, left outer join, EntityGraph, default_batch_fetch_size, batch size, N+1, 프록시 초기화, 면접 답변, interview answer, 조회 범위 통제]
topic: 연관관계를 왜 항상 LAZY 로 두는가 — LAZY 는 N+1 을 미룰 뿐이고, 해결은 fetch join. EAGER 는 쿼리 단위로 끌 수 없다는 비대칭
summary: LAZY 에서도 연관 데이터를 루프에서 읽으면 프록시 초기화로 N+1 이 그대로 생긴다(1+20=21번). 일반 join 은 select 절에 루트만 올려 프록시가 남으므로 join fetch(또는 @EntityGraph)로 한 번에 진짜 엔티티를 채워야 한다. fetch join 으로 LAZY 를 "이번만 켜는" 것은 가능하지만 EAGER 를 "이번만 끄는" 방법은 없어서, 쿼리마다 조회 범위를 통제하려고 기본값을 LAZY 로 둔다.
source: session
distilled: true
distilled_at: 2026-10-14
---

## 배운 개념

### 1. LAZY 인데도 N+1

```java
List<Post> posts = postRepository.findAll();   // 게시글 20개, 작성자 서로 다른 20명
for (Post post : posts) {
    System.out.println(post.getTitle() + " - " + post.getMember().getName());
}
```

- `findAll()` 1번 + `getName()` 에서 프록시 20개가 각각 초기화 20번 = **21번**.
- 결과만 보면 EAGER 와 같음 → **LAZY 는 N+1 을 없애지 않고 발생 시점을 뒤로 미룰 뿐**.

### 2. 일반 join 으로는 해결 안 됨

```java
@Query("select p from Post p left join p.member")
```

```sql
select p.id, p.title, p.member_id     -- member 컬럼 없음
from post p
left join member m on p.member_id = m.id
```

- 조인은 했지만 `select` 절에 **Post 컬럼만** 올라감 → `p.member` 는 여전히 **프록시** → 루프에서 또 N+1.
- `left` vs `inner` 는 **어떤 행을 가져오느냐**(작성자 없는 글 포함 여부)의 문제. **프록시가 남느냐**와는 별개.

### 3. fetch join

```java
@Query("select p from Post p left join fetch p.member")
List<Post> findAllWithMember();
```

- 조인 대상까지 `select` 절에 올려서 조회 시점에 **진짜 엔티티**로 채움 → 루프에서 추가 쿼리 없음 → **총 1번**.
- fetch 설정(EAGER/LAZY)으로는 조인 SQL 을 만들 수 없다(JPQL 은 그대로 번역) → 조인은 **JPQL 자체**에 써야 함.
- `member_id` 가 null 일 수 있으면 `left join fetch`.

### 4. 다른 선택지

| 방법 | 쿼리 수 | 메모 |
|---|---|---|
| `join fetch` | 1 | JPQL 에 직접 작성 |
| `@EntityGraph(attributePaths = "member")` | 1 | 어노테이션으로 fetch join |
| `default_batch_fetch_size` | 1+1 | 프록시들을 `where id in (...)` 로 한 번에 초기화. 컬렉션 + 페이징에 유용 |

### 5. 핵심 비대칭 → "왜 항상 LAZY?"

- LAZY 로 둔 연관 → 필요한 화면에서 fetch join 으로 **"이번만 함께 가져와"** 가능.
- EAGER 로 둔 연관 → 특정 쿼리에서 **"이번엔 가져오지 마"** 하는 방법이 **없음**.

### 6. 면접 답변 (30초)

> "EAGER 는 JPQL 로 조회할 때 쿼리가 그대로 번역된 뒤 연관 엔티티를 추가로 조회해서 N+1 이 생기고, 쓰지 않는 데이터까지 항상 가져옵니다. 게다가 특정 쿼리에서만 끄는 방법이 없습니다.
> 반대로 LAZY 는 프록시로 조회를 미뤄 두었다가, 연관 데이터가 필요한 화면에서만 fetch join 이나 `@EntityGraph` 로 한 번에 가져올 수 있습니다.
> LAZY 자체가 N+1 을 없애 주지는 않지만, **조회 범위를 쿼리마다 제가 통제할 수 있기 때문에** 기본값을 LAZY 로 두었습니다."

## 오답·헷갈린 점

- **전**: LAZY + 작성자 이름 출력 시 쿼리 **40번** → **후**: `getMember()` 는 쿼리 X, `getName()` 만 초기화 → 1 + 20 = **21번**. 스스로 정리한 "ID 외의 값을 읽을 때 조회" 기준을 적용하지 못함.
- **전**: 해결책 = "Post 엔티티에 작성자 이름도 조인에 건다" → **후**: 방향은 맞지만 구체성 부족. fetch 설정으로는 조인 못 만듦 → JPQL 에 **`join fetch`**.
- **전**: 빈칸 답 = "left outer join" (작성자 없는 글이 빠질 수 있어서) → **후**: outer join 시각 자체는 유효하지만, 일반 join 은 select 절에 Post 만 올려 **프록시가 남음**. 필요한 건 **fetch**.
- **전**: 최종 답변 첫 문장 "LAZY 로 하면 N+1 문제를 해결할 수 있고" → **후**: LAZY 는 N+1 을 **미룰 뿐**. 해결은 fetch join. 면접에서 이렇게 말하면 "작성자 이름 출력하면요?" 반격 들어옴.
- ⚠️ 반복 테마: "조인에 건다", "해결된다" 같은 뭉뚱그린 표현 → 구체 메커니즘(무엇이 select 절에 올라가나, 몇 번 나가나)으로 환원 필요.

## Q&A

- **Q. LAZY 에서 작성자 이름까지 출력하면 쿼리 몇 번?** A. 21번 (1 + 서로 다른 작성자 수).
- **Q. LAZY 로 바꾸면 N+1 이 해결되나?** A. 아니다. 시점만 미룸.
- **Q. 쿼리 1번으로 줄이려면?** A. `join fetch` / `@EntityGraph`. (batch size 는 1+1)
- **Q. 일반 join 으로는 왜 안 되나?** A. select 절에 루트 엔티티만 올라가서 연관은 프록시로 남음.
- **Q. fetch join 후 `p.member` 는?** A. 진짜 엔티티.
- **Q. EAGER 를 특정 쿼리에서만 끌 수 있나?** A. 없다. 그래서 기본은 LAZY.
- **Q. 왜 항상 LAZY 로 두었나?** A. 위 30초 답변.

## 다음 학습

- `LazyInitializationException`: 트랜잭션이 끝난 뒤 프록시 초기화 시 무슨 일이 생기나 + OSIV on/off 트레이드오프 → [[open-in-view-osiv]] (board 프로젝트에서 OSIV 끈 상태와 연결)
- 컬렉션(`@OneToMany`) fetch join + 페이징이 위험한 이유 (메모리 페이징, `HHH90003004` 경고)
- study-board 에서 게시글 목록 쿼리에 fetch join / `@EntityGraph` 실제 적용 여부 점검
- 연결: [[n-plus-1-problem]], [[lazy-loading-proxy]], [[2026-10-13-eager-jpql-n-plus-one]], [[2026-10-13-jpa-proxy-structure-equals]]
