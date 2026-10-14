---
type: refined
slug: denormalized-counter-atomic-update
tags: [반정규화, denormalization, 비정규화, 카운터컬럼, counter-column, 좋아요수, likeCount, 조회수, viewCount, 집계캐시, 카운트N+1, 정합성, consistency, 갱신손실, lost-update, 갱신유실, 동시성, concurrency, read-modify-write, 변경감지, dirty-checking, 원자적업데이트, atomic-update, like_count+1, 벌크연산, bulk-update, Modifying, clearAutomatically, flushAutomatically, 1차캐시, 낡은값, stale, 낙관적락, optimistic-lock, Version, 비관적락, pessimistic-lock, 행락, row-lock]
topic: 목록 count N+1 을 피하려고 카운터를 반정규화할 때 치르는 대가 — 정합성 관리 책임, 변경 감지 기반 +1 의 갱신 손실, 원자적 UPDATE + clearAutomatically
summary: 좋아요 수처럼 자식 행 개수를 부모 컬럼에 복사해 두면 목록에서 행마다 count 하는 N+1 은 사라지지만, 원본과 복사본을 같은 트랜잭션에서 직접 맞춰야 하는 정합성 책임이 생긴다. 엔티티에서 읽고 +1 하고 변경 감지로 쓰면 UPDATE 에 자바가 계산한 값이 들어가 동시 요청 시 갱신 손실(10→11)이 난다. DB 가 like_count + 1 을 하게 하는 원자적 UPDATE 로 바꾸고, 벌크 연산 뒤 1차 캐시의 낡은 값을 읽지 않도록 clearAutomatically 를 붙인다.
contributors: [dongju]
source_refs:
  - https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html
updated: 2026-10-14
---

# 반정규화 카운터와 원자적 UPDATE

## 개념 정의

목록에서 글마다 `count(*)` 를 하면 한 페이지에 count 쿼리가 행 수만큼 나간다 — 일종의 N+1 ([[n-plus-1-problem]]).
그래서 자식 행 개수를 부모에 **복사**해 둔다.

```java
/** 좋아요 수. 진실은 post_like 표에 있고 이 값은 목록에서 매번 세지 않으려고 함께 든다 */
@Column(nullable = false)
private int likeCount;
```

같은 사실이 **두 군데**에 있게 된다: 진실은 `post_like` 행 개수, `post.like_count` 는 복사본.

## 작동 원리

### 대가 ① — 정합성 관리 책임 (반정규화 자체의 대가)

DB 도 JPA 도 두 값을 **자동으로 맞춰 주지 않는다.** `post_like` 를 추가·삭제할 때마다 **같은 트랜잭션 안에서**
카운터도 직접 증감해야 한다. 한 군데라도 빠뜨리면(예: 회원 탈퇴로 `post_like` 일괄 삭제) 숫자가 실제와 어긋난 채
영원히 남는다.

### 대가 ② — read-modify-write 구현의 갱신 손실 (구현 방식의 대가)

```java
@Transactional
public boolean toggleLike(Long postId, Long memberId) {
    Post post = findById(postId);
    ...
    postLikeRepository.save(new PostLike(post, member));
    post.increaseLikeCount();          // this.likeCount++ → 커밋 시 변경 감지로 UPDATE
}
```

`@Version` 도 락도 없이 처음 값 10 에서 두 사용자가 동시에 누르면:

```
A: select → likeCount = 10 읽음
B: select → likeCount = 10 읽음      (A 가 아직 커밋 전)
A: 자바에서 10+1 → update post set like_count = 11 → 커밋
B: 자바에서 10+1 → update post set like_count = 11 → 커밋
최종: 11  → A 의 +1 이 B 에게 덮어써져 사라짐 = 갱신 손실(Lost Update)
```

핵심은 변경 감지 UPDATE 가 `set like_count = ?` 에 **자바가 계산한 값**을 넣는다는 점이다 ([[dirty-checking]]).
JVM 안의 `size++` 와 같은 read-modify-write 문제가 DB 행 위에서 일어난 것이다 ([[race-condition-lost-update]]).

### 해결 — DB 가 직접 +1 하게 한다

```sql
update post set like_count = like_count + 1 where id = ?
```

DB 가 **그 순간의 현재 값**에 +1 한다. 동시에 실행돼도 행 락으로 순서가 서서 10 → 11 → 12.

```java
@Modifying(clearAutomatically = true)
@Query("update Post p set p.likeCount = p.likeCount + 1 where p.id = :id")
int increaseLikeCount(@Param("id") Long id);
```

### 벌크 연산이므로 clear 가 따라온다

`@Modifying @Query` 는 영속성 컨텍스트를 거치지 않고 DB 로 바로 간다. 같은 트랜잭션에서 이미 `findById` 한 `post` 가
1차 캐시에 있으면 **DB 는 11 인데 `post.getLikeCount()` 는 10** 이다 ([[jpa-bulk-update-and-clear]]).

| 옵션 | 방향 | 해결하는 문제 |
|---|---|---|
| `flushAutomatically` | 영속성 컨텍스트 → DB | 벌크 **전에** 아직 안 나간 변경을 먼저 보냄 |
| `clearAutomatically` | 1차 캐시 비움 | 벌크 **후에** 낡은 값(10)을 읽지 않게 함 |

"DB 는 최신, 캐시는 낡음" 이므로 **clear** 다. flush 는 반대 방향이라 이 문제를 풀지 못한다 ([[jpa-flush]]).

## 트레이드오프 / 한계

| 방법 | 동작 | 대가 |
|---|---|---|
| 원자적 UPDATE | DB 가 현재 값에 +1 | 벌크 연산 → 1차 캐시 clear 필요, 엔티티 메서드로 표현 못 함 |
| 낙관적 락 `@Version` | 커밋 시 버전 불일치면 예외 | 충돌 시 **재시도 로직** 필요, 인기 글엔 충돌 잦음 |
| 비관적 락 `PESSIMISTIC_WRITE` | `select ... for update` 로 행 잠금 | 대기 → 처리량 저하, 데드락 주의 |
| 반정규화 안 함 | 매번 `count(*)` | 목록 쿼리 비용 (인덱스·캐시로 완화 가능) |

단순 증감 카운터라면 원자적 UPDATE 가 가장 가볍다. 감소는 `greatest(like_count - 1, 0)` 이나 `where like_count > 0` 으로
음수를 막는다.

> **면접 답변** — "좋아요 수는 목록에서 글마다 count 를 하지 않으려고 Post 에 반정규화해 두었습니다. 대신 좋아요 행과 함께
> 갱신할 책임이 생겼고, 지금은 엔티티에서 +1 하는 방식이라 동시 요청 시 갱신 손실 위험이 있어 DB 원자적 UPDATE 로 바꿀 계획입니다."
> — 스스로 찾은 약점과 개선 계획을 먼저 말하면 지적거리를 강점으로 바꿀 수 있다.

> [!WARNING]
> **오답 코너**
> - **"반정규화의 대가는 갱신 손실"** — 본질적 대가는 **정합성 관리 책임**이다. 갱신 손실은 read-modify-write **구현 방식**의 문제.
> - **"갱신 지연"** — 정확한 용어는 **갱신 손실(Lost Update)**. 지연이 아니라 한쪽 변경이 **사라진다**.
> - **"벌크 UPDATE 후 낡은 값 문제는 `flushAutomatically` 로 푼다"** — flush 는 컨텍스트 → DB 방향. 낡은 캐시를 버리는 건 **`clearAutomatically`**.
> - **"`@Transactional` 이 있으니 동시 +1 도 안전하다"** — 트랜잭션은 원자성·격리를 주지만, 기본 격리 수준에서 두 트랜잭션이
>   같은 값을 읽고 각자 계산한 값으로 덮어쓰는 것은 막지 못한다.

## 복습 체크

- [ ] 카운터를 반정규화하는 이유를 N+1 과 연결해 말할 수 있는가?
- [ ] 반정규화의 본질적 대가와, 구현 방식에서 오는 대가를 구분할 수 있는가?
- [ ] 처음 값 10, 동시 좋아요 2번에서 두 트랜잭션의 읽기·UPDATE 를 순서대로 적고 결과가 11 인 이유를 말할 수 있는가?
- [ ] 원자적 UPDATE SQL 을 한 줄로 쓰고, 왜 동시에 실행돼도 12 가 되는지 설명할 수 있는가?
- [ ] 벌크 UPDATE 직후 같은 트랜잭션에서 엔티티의 카운터를 읽으면 몇이 나오며, 어떤 옵션으로 막는가?
- [ ] `flushAutomatically` 와 `clearAutomatically` 의 방향 차이는?
- [ ] 원자적 UPDATE / 낙관적 락 / 비관적 락의 대가를 비교할 수 있는가?

## 관련

[[n-plus-1-problem]] · [[jpa-bulk-update-and-clear]] · [[jpa-flush]] · [[dirty-checking]] · [[race-condition-lost-update]] · [[transactional-rollback-rules]]
