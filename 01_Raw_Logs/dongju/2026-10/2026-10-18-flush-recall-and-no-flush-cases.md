---
date: 2026-10-18
type: raw
author: dongju
tags: [플러시, flush, 커밋, commit, flush vs commit, 자동 플러시, auto flush, FlushModeType.AUTO, FlushMode.MANUAL, readOnly, 변경 감지, dirty checking, 쓰기 지연, write-behind, 1차 캐시, first-level cache, clear, flush vs clear, JPQL 직전 플러시, em.find, 롤백, rollback, IDENTITY, 즉시 INSERT, 복습, recall, 면접 답변]
topic: 10/12 flush vs commit 의 6일 뒤 복습 — 방향은 고쳐졌고 시점·clear 구분이 약점. 확장: 플러시가 일어나지 않는 곳(find, readOnly=MANUAL, 롤백)과 IDENTITY 즉시 INSERT 는 플러시가 아니라는 점
summary: 플러시는 변경 감지로 UPDATE 를 만들고 쓰기 지연 저장소의 SQL 을 영속성 컨텍스트에서 DB 로 보내는 동기화이고, 커밋은 그 내용을 확정한다. 플러시 후 커밋 전 예외면 롤백되며 1차 캐시는 그대로 남는다(비우는 건 clear, clear 는 오히려 미플러시 변경을 버린다). 자동 플러시는 커밋 직전과 JPQL 실행 직전이고, 후자는 DB 만 보는 JPQL 이 영속성 컨텍스트에만 있는 엔티티를 놓치지 않게 하려는 것이다. em.find 는 1차 캐시를 먼저 봐서, readOnly 는 FlushMode.MANUAL 이라서, 롤백 종료는 보낼 이유가 없어서 플러시하지 않는다. IDENTITY 의 persist 즉시 INSERT 는 변경 감지 없이 그 한 건만 실행하는 것이라 플러시가 아니며, 다른 엔티티의 UPDATE 는 커밋 직전에 나간다.
source: session
distilled: false
---

## 배운 개념

### 1. 플러시 vs 커밋 (복습)

```
플러시
 ① 변경 감지 — 1차 캐시 엔티티와 스냅샷 비교 → UPDATE 생성 (setTitle 만으로는 SQL 조차 없음)
 ② 쓰기 지연 저장소의 INSERT / UPDATE / DELETE 를 영속성 컨텍스트 → DB 로 전송
커밋
 → 보낸 내용을 확정
```

- 플러시 후 커밋 전 예외 → **롤백**, DB 에 남지 않음.
- 플러시 후 1차 캐시 → **그대로** (엔티티는 계속 영속). 비우는 건 `clear()`.

```java
@Transactional
public void update(Long id) {
    Post post = postRepository.findById(id).get();    // 영속
    postRepository.findSummaries(...);                // JPQL → 직전에 자동 플러시
    post.setTitle("새 제목");                           // 플러시가 캐시를 비웠다면 post 는 준영속 → 수정 증발 (모순)
}
```

- clear 는 플러시하지 않은 변경을 **버린다** → 대량 배치는 flush → clear 순서 ([[jpa-bulk-update-and-clear]]).

### 2. 자동 플러시 시점 (`FlushModeType.AUTO`)

1. **커밋 직전** (롤백으로 끝나면 안 함)
2. **JPQL 실행 직전**

```java
em.persist(memberA);                                           // 엔티티는 1차 캐시, INSERT 는 쓰기 지연 저장소
em.createQuery("select m from Member m", Member.class).getResultList();
// JPQL 은 1차 캐시를 거치지 않고 DB 로 바로 → 플러시 안 하면 memberA 가 결과에서 빠짐
```

> JPQL 은 DB 만 보는데 방금 persist 한 엔티티는 영속성 컨텍스트에만 있으므로, 플러시하지 않으면 조회 결과에서 빠진다.

### 3. 플러시가 일어나지 않는 곳

| 상황 | 이유 |
|---|---|
| `em.find()` 직전 | 1차 캐시를 먼저 봄 → 방금 persist 한 것도 캐시에서 찾음 → DB 와 맞출 필요 없음 |
| `@Transactional(readOnly = true)` | Hibernate `FlushMode.MANUAL`(자동 플러시 없음, `em.flush()` 직접 호출해야) + 스냅샷 생략 → `setTitle()` 이 **예외 없이 조용히 무시** |
| 롤백으로 끝나는 트랜잭션 | 보낼 이유 없음 |

### 4. IDENTITY 즉시 INSERT 는 플러시가 아니다

```java
@Transactional
public void write(Long postId) {
    Post post = postRepository.findById(postId).get();
    post.setTitle("새 제목");          // ① 변경
    em.persist(new Comment(...));     // ② IDENTITY → INSERT 즉시 실행
}                                     // ③ 커밋 직전 플러시 → post 의 UPDATE 는 여기서
```

| | IDENTITY `persist` (②) | 플러시 (③) |
|---|---|---|
| 변경 감지 | ❌ | ✅ |
| 보내는 것 | 그 엔티티의 INSERT **한 건** (쓰기 지연 저장소에 쌓이지도 않음) | 쌓인 SQL **전부** |
| 이유 | id 를 받아 1차 캐시에 넣으려고 | 전체 동기화 |

- ②가 플러시였다면 ①의 UPDATE 도 그 순간 함께 나갔어야 한다 → 실제로는 커밋까지 기다림 → 플러시 아님.

### 5. 면접 답변 (30초)

> "플러시는 영속성 컨텍스트의 변경 내용을 DB에 보내는 동기화 작업입니다. 먼저 스냅샷과 비교하는 변경 감지로 UPDATE를 만들고, 쓰기 지연 저장소에 쌓인 SQL을 영속성 컨텍스트에서 DB로 보냅니다. 커밋은 그렇게 보낸 내용을 확정하는 것이라, 플러시 후라도 커밋 전에 예외가 나면 롤백됩니다. 플러시는 보내기만 할 뿐 1차 캐시는 그대로 남고, 비우는 건 clear입니다.
> 자동 플러시는 커밋 직전과 JPQL 실행 직전에 일어납니다. JPQL은 1차 캐시를 거치지 않고 DB만 조회하는데, 방금 persist한 엔티티는 영속성 컨텍스트에만 있으니 먼저 보내지 않으면 조회 결과에서 빠지기 때문입니다.
> 반대로 em.find는 1차 캐시를 먼저 보니 플러시하지 않고, readOnly 트랜잭션은 FlushMode가 MANUAL이라 자동 플러시가 없습니다. 참고로 IDENTITY 엔티티를 persist할 때 INSERT가 즉시 나가는 건 플러시가 아니라 그 한 건만 먼저 실행하는 것이라, 다른 엔티티의 UPDATE는 여전히 커밋 직전에 나갑니다."

## 오답·헷갈린 점

### 📈 10/12 대비

| | 10/12 | 10/18 |
|---|---|---|
| 플러시의 방향 | ❌ "영속성 컨텍스트에 올리는" | ✅ 영속성 컨텍스트 → DB |
| 자동 플러시 시점 | ❌ "트랜잭션의 시작과 커밋" | ❌ → ✅ (힌트 후) |
| JPQL 직전 플러시 이유 | ❌ 자기모순 | ✅ 스스로 설명 |

### 오늘의 오답

- **전**: 플러시 후 1차 캐시는 "초기화한다" → **후**: **그대로**. 비우는 건 `clear()`.
- **전**: 자동 플러시 시점 = "트랜잭션의 종료" → **후**: **커밋 직전** (롤백 종료 시엔 안 함).
- **전**: 자동 플러시 시점 = "clear 시" → **후**: **반대**. clear 는 미플러시 변경을 버림.
- **전**: readOnly 는 플러시를 "읽기 전용으로 변경" → **후**: **`FlushMode.MANUAL`**.
- **전**: IDENTITY persist 의 즉시 INSERT = "플러시다", "쌓인 SQL 의 전송이기 때문" → **후**: 플러시 **아님**. 변경 감지 없이 그 한 건만 즉시(쌓이지도 않음). 결론(UPDATE 는 커밋 직전)과 자기모순이었음.
- **전**: 최종 답변 "플러시는 변경감지와 쓰기 지연이 있다" → **후**: 동작과 **방향**으로 — UPDATE 를 **만들고**, SQL 을 **영속성 컨텍스트 → DB 로 보낸다**. 커밋과의 대비("확정")도 넣을 것.
- ⚠️ **flush / clear 혼동 — 오늘만 3회** (`clearAutomatically` 설명 × 2, 이번 인터뷰). `jpa-flush` + `jpa-bulk-update-and-clear` 를 나란히 study-review 할 것.

## Q&A

- **Q. 플러시 한 문장 (방향 포함)?** A. 영속성 컨텍스트의 쌓인 SQL 을 DB 로 보내는 것 (+ 그 전에 변경 감지로 UPDATE 생성).
- **Q. 플러시 후 커밋 전 예외면?** A. 롤백, 남지 않음.
- **Q. 플러시 후 1차 캐시?** A. 그대로.
- **Q. 자동 플러시 시점 두 가지와 JPQL 직전 이유?** A. 커밋 직전, JPQL 실행 직전. JPQL 은 DB 만 봄.
- **Q. em.find 직전 플러시?** A. 안 함. 1차 캐시 먼저.
- **Q. readOnly 에서 setTitle?** A. FlushMode.MANUAL + 스냅샷 생략 → 조용히 무시.
- **Q. IDENTITY 즉시 INSERT 는 플러시?** A. 아님. 다른 UPDATE 는 커밋 직전.

## 다음 학습

- `FlushModeType.COMMIT` vs AUTO, Hibernate 의 "쿼리 대상 테이블과 관련 있을 때만 플러시" 최적화
- Native Query 실행 직전 플러시 동작 (JPA EntityManager API vs Hibernate 고유 API)
- `show-sql` 로 JPQL 직전 자동 플러시와 IDENTITY 즉시 INSERT 로그 순서 직접 확인
- 연결: [[jpa-flush]], [[jpa-bulk-update-and-clear]], [[dirty-checking]], [[jpa-id-generation-strategy]]
