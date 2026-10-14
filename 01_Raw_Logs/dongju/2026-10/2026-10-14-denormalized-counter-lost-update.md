---
date: 2026-10-14
type: raw
author: dongju
tags: [반정규화, denormalization, 카운터 컬럼, counter column, likeCount, 좋아요 수, 정합성, consistency, 갱신 손실, lost update, 동시성, concurrency, read-modify-write, 변경 감지, dirty checking, 원자적 업데이트, atomic update, 벌크 연산, bulk update, Modifying, clearAutomatically, flushAutomatically, 1차 캐시, first-level cache, 낙관적 락, optimistic lock, Version, 비관적 락, pessimistic lock, study-board]
topic: 목록 count N+1 을 피하려고 좋아요 수를 반정규화하면 치르는 대가 — 정합성 관리 책임과, 변경 감지 기반 +1 의 갱신 손실, 그리고 원자적 UPDATE + clearAutomatically
summary: 반정규화한 카운터는 원본(post_like 행)과 같은 트랜잭션에서 개발자가 직접 맞춰야 하는 정합성 책임이 생긴다. 여기에 엔티티에서 읽고 +1 하고 변경 감지로 쓰는 구현은 동시 요청 시 갱신 손실(10→11)이 생기므로, DB 가 직접 like_count + 1 을 하는 원자적 UPDATE 로 바꾸고 벌크 연산 뒤 1차 캐시의 오래된 값을 읽지 않도록 @Modifying(clearAutomatically = true) 를 붙인다.
source: session
distilled: true
distilled_at: 2026-10-14
---

## 배운 개념

### 1. 왜 반정규화했나 — 이것도 일종의 N+1

```java
/** 좋아요 수. 진실은 post_like 표에 있고 이 값은 목록에서 매번 세지 않으려고 함께 든다 */
@Column(nullable = false)
private int likeCount;
```

- 목록에서 글마다 `count(*)` → 한 페이지에 count 쿼리 20번. 그래서 숫자를 `Post` 에 **복사**.

### 2. 대가 ① — 정합성 관리 책임

- 같은 사실이 **두 군데**: 진실은 `post_like` 행 개수, `post.like_count` 는 복사본.
- DB 도 JPA 도 자동으로 맞춰 주지 않는다 → `post_like` 추가·삭제 때마다 **같은 트랜잭션에서** 카운터도 직접 증감.
- 한 군데라도 빠뜨리면(예: 회원 탈퇴로 post_like 일괄 삭제) 숫자가 실제와 어긋난 채로 영원히 남음.

### 3. 대가 ② — 구현 방식에 따른 갱신 손실

```java
// PostService.toggleLike (study-board 실제 코드)
@Transactional
public boolean toggleLike(Long postId, Long memberId) {
    Post post = findById(postId);
    ...
        postLikeRepository.save(new PostLike(post, memberRepository.getReferenceById(memberId)));
        post.increaseLikeCount();          // this.likeCount++ → 변경 감지로 UPDATE
```

`@Version` 도 락도 없음. 처음 값 10, 두 사용자가 동시에 좋아요:

```
A: select → likeCount = 10 읽음
B: select → likeCount = 10 읽음      (A 가 아직 커밋 전)
A: 자바에서 10+1 → update post set like_count = 11 → 커밋
B: 자바에서 10+1 → update post set like_count = 11 → 커밋
최종: 11  → A 의 변경이 B 에게 덮어써져 사라짐 = 갱신 손실(Lost Update)
```

- 핵심: 변경 감지 UPDATE 는 `set like_count = ?` 에 **자바가 계산한 값**을 넣는다.
- 갱신 손실은 반정규화 **자체**가 아니라 **read-modify-write 구현**의 문제.

### 4. 해결 — DB 가 직접 +1

```sql
update post set like_count = like_count + 1 where id = ?
```

- DB 가 **그 순간의 현재 값**에 +1. 동시에 실행돼도 행 락으로 순서가 서서 10 → 11 → 12.

```java
@Modifying(clearAutomatically = true)
@Query("update Post p set p.likeCount = p.likeCount + 1 where p.id = :id")
int increaseLikeCount(@Param("id") Long id);
```

### 5. 벌크 연산의 주의사항 — clear 가 필요한 이유

- `@Modifying @Query` 는 영속성 컨텍스트를 거치지 않고 DB 로 바로 간다.
- 같은 트랜잭션에서 이미 `findById` 한 `post` 가 1차 캐시에 남아 있으면 → DB 는 11 인데 `post.getLikeCount()` 는 **10**.

| 옵션 | 방향 | 해결하는 문제 |
|---|---|---|
| `flushAutomatically` | 영속성 컨텍스트 → DB | 벌크 **전에** 아직 안 나간 변경을 먼저 보냄 |
| `clearAutomatically` | 1차 캐시 비움 | 벌크 **후에** 오래된 값(10)을 읽지 않게 함 |

- 지금 문제는 "DB 는 최신, 캐시는 낡음" → **clear**. flush 는 반대 방향이라 못 푼다.

### 6. 면접 답변 (마무리 문장)

> "좋아요 수는 목록에서 글마다 count 를 하지 않으려고 Post 에 반정규화해 두었습니다. 대신 좋아요 행과 함께 갱신할 책임이 생겼고, 지금은 엔티티에서 +1 하는 방식이라 동시 요청 시 갱신 손실 위험이 있어 DB 원자적 UPDATE 로 바꿀 계획입니다."

- 스스로 찾은 약점 + 개선 계획을 먼저 말하면, 면접관이 지적할 거리를 강점으로 바꿀 수 있다.

## 오답·헷갈린 점

- **전**: 반정규화 후 post_like 변경 시 해야 할 일 = "모르겠다" → **후**: 같은 트랜잭션에서 카운터도 직접 증감. DB/JPA 가 자동으로 맞춰 주지 않음.
- **전**: "갱신 지연" → **후**: **갱신 손실(Lost Update)**. 지연이 아니라 A 의 +1 이 사라짐.
- **전**: 벌크 UPDATE 후 오래된 값 문제 → "flush 옵션" → **후**: **`clearAutomatically = true`**. flush 는 영속성 컨텍스트 → DB 방향이라 반대.
- **전**: 반정규화의 대가 = "갱신 손실" → **후**: 본질적 대가는 **정합성 관리 책임**. 갱신 손실은 read-modify-write 구현 방식의 문제.
- ⚠️ 10/12 벌크 연산·clear 세션과 flush 방향 세션을 배웠는데 flush/clear 를 다시 혼동 → [[jpa-bulk-update-and-clear]], [[jpa-flush]] 복습.

## Q&A

- **Q. likeCount 를 왜 Post 에 두었나?** A. 목록에서 글마다 count 하는 N+1 을 피하려고.
- **Q. 반정규화의 대가?** A. 원본과 복사본을 같은 트랜잭션에서 직접 맞춰야 하는 정합성 책임.
- **Q. 처음 10, 동시 좋아요 2번 → 결과?** A. 11. 갱신 손실.
- **Q. 원자적 UPDATE SQL?** A. `update post set like_count = like_count + 1 where id = ?`
- **Q. 벌크 UPDATE 직후 같은 트랜잭션에서 getLikeCount()?** A. 10 (1차 캐시). `clearAutomatically = true` 로 해결.

## 다음 학습

- 🔧 study-board `PostService.toggleLike` 를 원자적 UPDATE 로 리팩터링 + 동시성 테스트(`ExecutorService` + `CountDownLatch`) 작성 (TDD 하드룰)
- 원자적 UPDATE vs 낙관적 락(`@Version` + 재시도) vs 비관적 락(`PESSIMISTIC_WRITE`) 비교
- `decreaseLikeCount` 의 `Math.max(0, ...)` 를 SQL 로 옮기면? (`greatest(like_count - 1, 0)` / `where like_count > 0`)
- `viewCount++` 도 같은 문제가 있는지 점검
- 연결: [[jpa-bulk-update-and-clear]], [[dirty-checking]], [[2026-10-14-n-plus-one-definition-solutions]]
