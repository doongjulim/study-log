---
date: 2026-08-25
type: raw
author: dongju
tags: [open-in-view, OSIV, osiv, spring.jpa.open-in-view, 영속성-컨텍스트, persistence-context, EntityManager, 지연로딩, lazy-loading, LazyInitializationException, 커넥션풀, connection-pool, HikariCP, 커넥션-고갈, pool-exhaustion, 변경감지, dirty-checking, flush, fetch-join, EntityGraph, N+1, 준영속, detached, JPA, Hibernate]
topic: spring.jpa.open-in-view 를 false 로 끄는 이유 — 커넥션 점유 구간과 의도치 않은 UPDATE
summary: OSIV 는 영속성 컨텍스트를 뷰 렌더링까지 열어둬 지연 로딩 편의를 주지만, 그 대가로 DB 커넥션을 요청 끝까지 점유하고 트랜잭션 밖 엔티티 변경이 후속 트랜잭션의 flush 에 딸려 나간다. 끄는 대신 fetch join / DTO 조회로 트랜잭션 안에서 데이터를 확정 짓는다.
source: session
distilled: true
distilled_at: 2026-08-25
---

## 배운 개념

### 1. `open-in-view` 가 바꾸는 것은 "영속성 컨텍스트의 생명주기"

| 설정 | 영속성 컨텍스트(Hibernate Session) 생존 범위 |
|---|---|
| `true` (Spring Boot 기본값) | 요청 시작 ~ **뷰 렌더링 완료**까지 |
| `false` | `@Transactional` 시작 ~ 종료까지 |

핵심 뉘앙스: **`true` 여도 트랜잭션 자체는 서비스 계층에서 시작하고 끝난다.**
즉 `true` 에는 **"트랜잭션은 이미 커밋됐는데 영속성 컨텍스트만 살아있는 구간"** 이 생기고,
OSIV 의 모든 문제(그리고 모든 편의)는 정확히 이 구간에서 발생한다.

### 2. OSIV 가 기본값 `true` 인 이유 — `LazyInitializationException` 회피

```java
// Service
@Transactional(readOnly = true)
public Post findPost(Long id) {
    return postRepository.findById(id).orElseThrow();
}
```

```html
<!-- Thymeleaf -->
<span th:text="${post.member.nickname}">  <!-- member 는 LAZY -->
```

`open-in-view=false` 면 이 화면에서 `LazyInitializationException`
(`could not initialize proxy - no Session`) 이 터진다.

> 프록시가 실제 값을 채우려고 **Session 에 쿼리를 부탁**했는데, 그 Session 이 이미 닫혀 있어서 발생.
> DB 도 커넥션도 멀쩡하다. 없는 건 오직 **부탁할 대상(Session)** 이다.

→ OSIV 를 켜두면 서비스 계층에서 미리 fetch join / DTO 변환을 하지 않아도
컨트롤러·뷰에서 연관 엔티티를 자유롭게 탐색할 수 있다. **개발 편의성**이 존재 이유.

### 3. 끈 이유 ① — DB 커넥션 풀 고갈

요청 하나의 타임라인:

```
① 요청 도착
② 컨트롤러 진입
③ 서비스 @Transactional 시작 → 조회        ← 여기서 커넥션 획득 (lazy acquisition)
④ 서비스 @Transactional 커밋/종료           ← 트랜잭션은 끝났지만 커넥션은 안 놓음
⑤ 컨트롤러 복귀, 외부 결제 API 호출 (3초)   ← DB 쿼리 0건인데 커넥션 점유 중
⑥ 뷰 렌더링 (지연 로딩 발생)
⑦ 응답 완료                                 ← 여기서 EntityManager close → 커넥션 반납
```

- **커넥션 점유 구간 = ③(첫 쿼리) ~ ⑦(요청 완료)**
- 반납 시점이 ⑥ 이 아니라 ⑦ 인 이유: ⑥ 은 아직 지연 로딩이 한창 일어나는 중이다.
  실제로는 스프링의 `OpenEntityManagerInViewInterceptor` 가
  **뷰 렌더링까지 끝난 뒤 `afterCompletion`** 에서 EntityManager 를 닫는다.
- ④ 에서 커넥션을 못 놓는 이유: **⑥ 에서 언제 지연 로딩이 튀어나올지 모르기 때문.**
  → **"영속성 컨텍스트가 살아있다" ≒ "커넥션도 붙잡고 있다"**

**장애 시나리오 (풀 크기 10, 초당 20 요청)**

1. 11번째 요청부터 커넥션을 못 받고 **큐에서 대기(blocking)**
2. 앞 요청들은 ⑤ 의 3초 외부 API 를 기다리느라 커넥션을 안 놓음
3. 대기 시간이 `connection-timeout`(HikariCP 기본 30초) 초과
4. `SQLTransientConnectionException: Connection is not available, request timed out`
5. **커넥션 풀 고갈(pool exhaustion)** + 스레드 점유 → 서비스 마비
6. 장애가 격리되지 않고 **같은 풀을 공유하는 전체 서비스로 전파**
   (게시판 상세 하나 때문에 로그인·글쓰기도 동반 사망)

### 4. 끈 이유 ② — 의도치 않은 UPDATE (무결성)

```java
@GetMapping("/posts/{id}")
public String view(@PathVariable Long id, Model model) {
    Post post = postService.findPost(id);   // 영속 상태로 반환
    post.setTitle("컨트롤러에서 몰래 수정");   // ← 트랜잭션 밖
    someOtherService.doSomething();          // ← 여기서 새 @Transactional 시작/커밋
    ...
}
```

인과 사슬:

1. dirty checking 은 "값을 바꾸는 순간"이 아니라 **flush 순간**에 스냅샷과 비교해 동작한다.
2. `open-in-view=true` 라 요청 내내 영속성 컨텍스트는 **딱 하나**.
   `post` 는 여전히 **영속(managed)** 상태 → 스냅샷 비교 대상.
3. `someOtherService` 가 새 트랜잭션을 커밋하며 flush → 그 flush 는
   **컨텍스트 안의 모든 영속 엔티티**를 훑는다. `post` 를 건드린 적도 없는데.
4. → **아무도 의도하지 않은 `UPDATE post SET title=...` 이 DB 로 나간다.**

무서운 이유: **변경한 코드(컨트롤러)와 실제로 DB 에 반영시킨 코드(무관한 다른 서비스)가
완전히 분리되어 있어** 장애 원인 추적이 지옥이 된다.

`false` 면 트랜잭션 종료와 함께 영속성 컨텍스트가 닫히고 `post` 는 **준영속(detached)** 이 되므로,
아무리 값을 바꿔도 flush 가 쳐다보지 않는다 → **원천 봉쇄**.

### 5. 끈 대가 — 트랜잭션 안에서 데이터를 확정 짓기

| 계열 | 방법 |
|---|---|
| 트랜잭션 안에서 미리 채우기 | `join fetch`(JPQL), `@EntityGraph`, `default_batch_fetch_size` |
| 애초에 엔티티를 안 넘기기 | DTO 직접 조회(projection), 서비스에서 DTO 변환 후 반환 |

```java
// 1) JPQL — join fetch
@Query("select p from Post p join fetch p.member")
List<Post> findAllWithMember();

// 2) Spring Data JPA — @EntityGraph
@EntityGraph(attributePaths = {"member"})
List<Post> findAll();
```

주의: 그냥 트랜잭션 안에서 for 문으로 프록시를 하나씩 초기화하면 **N+1 문제**.
글 100건이면 목록 1번 + 작성자 100번 = **쿼리 101번**.

> **결론: `open-in-view=false` 는 "지연 로딩이 트랜잭션 밖으로 새어나가지 못하게 하고,
> 필요한 데이터를 트랜잭션 안에서 확정 짓겠다"는 설계 선언이다.**
> 커넥션 절약과 무결성 확보는 그 선언의 결과물이다.

## 오답·헷갈린 점

1. **`true`/`false` 의미를 정반대로 알고 있었음** (가장 큰 오개념)
   - 전: `false` 일 때 영속성 컨텍스트가 요청~뷰까지 유지된다
   - 후: **`true` 가 뷰까지, `false` 가 트랜잭션 범위.**
     옵션 이름 자체가 "view 에서도 열어둔다(open-in-view)" 라는 뜻이다.

2. **OSIV 의 존재 이유를 "데이터 무결성" 으로 알고 있었음**
   - 전: 무결성이 깨질 수 있어서 OSIV 가 필요하다
   - 후: 무결성 문제는 **OSIV 를 켰을 때 생기는 부작용** 쪽이다.
     존재 이유는 **뷰 계층에서의 지연 로딩 편의(= LazyInitializationException 회피)**.

3. **끄면 아끼는 자원을 "메모리" 로만 인식**
   - 전: 메모리를 아낄 수 있다
   - 후: 메모리는 부수적. 진짜는 **DB 커넥션의 조기 반납**. 커넥션은 pool 로 개수가 고정되어 있어
     고갈되면 서비스 전체가 죽는다.

4. **커넥션 반납 시점을 ⑥(뷰 렌더링) 으로 답함**
   - 전: 뷰 렌더링 시점에 반납
   - 후: **⑦ 렌더링 완료 후(`afterCompletion`).** 렌더링 도중에도 지연 로딩이 발생하므로
     그때 반납하면 또 `LazyInitializationException`.
   - 획득 시점도 ① 이 아니라 **③ 첫 쿼리 시점(lazy acquisition)**.

5. **"커넥션 오버플로우" 라는 용어 사용**
   - 전: 커넥션 오버플로우가 발생한다
   - 후: 풀은 **넘치지 않는다.** 크기가 10이면 11번째는 만들어지지 않는다.
     정확한 용어는 **고갈(exhaustion) + 대기(blocking) + 타임아웃**.

6. **`LazyInitializationException` 의 원인을 "DB 를 건드려서" 로 설명**
   - 전: 영속성 컨텍스트가 죽은 상태에서 DB 를 건드려서 터진다
   - 후: **DB 를 건드리려고 "부탁할 대상(Session)" 이 없어서** 터진다.
     DB 와 커넥션은 멀쩡하다.

7. **컨트롤러에서의 엔티티 변경은 DB 에 반영 안 된다고 답함**
   - 전: 트랜잭션 밖이니 변경되지 않는다
   - 후: **반영될 수 있다.** 영속 상태 유지 + 후속 트랜잭션의 flush 조합 때문.
     인과의 주어는 "업데이트" 가 아니라 **"뒤에 열린 남의 트랜잭션"**.

## Q&A

**Q. `open-in-view=true` 일 때 트랜잭션도 뷰까지 늘어나나?**
A. 아니다. 트랜잭션은 여전히 `@Transactional` 범위에서 끝난다.
늘어나는 건 **영속성 컨텍스트(EntityManager)의 생명주기**뿐이다.
그래서 "트랜잭션은 끝났는데 컨텍스트만 살아있는 구간" 이 생긴다.

**Q. 트랜잭션이 끝났는데 왜 커넥션을 못 놓나?**
A. 뷰 렌더링 중 언제든 지연 로딩이 발생해 `SELECT` 를 날려야 할 수 있기 때문.
④ 에서 반납해버리면 ⑥ 에서 쿼리를 날릴 수단이 없다.

**Q. 외부 API 호출 3초가 왜 위험한가?**
A. 그 3초 동안 **DB 쿼리는 단 한 줄도 안 나가는데 커넥션은 점유**되어 있다.
순수 낭비이며, 동시 요청이 풀 크기를 넘는 순간 대기 → 타임아웃 → 고갈로 이어진다.

**Q. 커넥션 풀이 고갈되면 이 API 하나만 죽나?**
A. 아니다. **같은 DB 커넥션 풀을 공유하는 전체 서비스**가 같이 죽는다. 장애가 격리되지 않는다.

**Q. `false` 면 컨트롤러의 엔티티 수정이 왜 DB 에 안 나가나?**
A. 트랜잭션 종료 시 영속성 컨텍스트가 닫히면서 엔티티가 **준영속(detached)** 이 된다.
준영속 엔티티는 스냅샷 비교 대상이 아니므로 flush 가 무시한다.

**Q. `join fetch` 와 `@EntityGraph` 의 차이는?**
A. 하는 일은 같다 — 트랜잭션 안에서 한 방의 조인 쿼리로 연관 엔티티까지 미리 채워온다.
JPQL 직접 작성이냐, Spring Data JPA 어노테이션이냐의 차이.

## 다음 학습

- `default_batch_fetch_size` — fetch join 으로 다 못 푸는 컬렉션 다중 조인(카테시안 곱) 문제와 IN 절 배치 로딩
- fetch join 의 한계: 컬렉션 fetch join 시 페이징 불가(`firstResult/maxResults` 경고, 메모리 페이징)
  → 이전 raw `2026-07-11-pageable-paging-search` 와 연결해서 볼 것
- DTO 직접 조회 vs 엔티티 조회 후 변환 — 성능/유지보수 트레이드오프
- HikariCP 풀 사이징 공식과 `connection-timeout`, `leak-detection-threshold` 튜닝
- 영속성 컨텍스트 1차 캐시·쓰기 지연 SQL 저장소·스냅샷의 내부 구조 (dirty checking 심화)
