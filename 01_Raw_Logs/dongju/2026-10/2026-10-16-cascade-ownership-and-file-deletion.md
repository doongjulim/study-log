---
date: 2026-10-16
type: raw
author: dongju
tags: [cascade, orphanRemoval, 생명주기, lifecycle, 단일 소유, single owner, ManyToOne cascade, 회원 삭제, FK 제약조건, foreign key constraint, DataIntegrityViolationException, 롤백, rollback, 파일 삭제, file deletion, 디스크 파일, 파일 시스템 롤백 불가, TransactionalFileRemover, removeAfterCommit, TransactionSynchronization, afterCommit, OrphanFileCleanupScheduler, 고아 파일, TransactionalEventListener, AFTER_COMMIT, 면접 답변, study-board]
topic: cascade·orphanRemoval 을 거는 기준(부모가 자식 생명주기를 단독 소유)과 @ManyToOne 에 걸면 생기는 사고, 그리고 JPA 가 지우지 못하는 디스크 파일을 커밋 후에 지우는 이유
summary: cascade·orphanRemoval 은 부모가 자식의 생명주기를 단독으로 소유할 때만 건다. Post.author 같은 @ManyToOne 에 cascade 를 걸면 글 삭제가 회원 DELETE 로 전이되어, 다른 글이 있으면 FK 위반으로 글 삭제까지 롤백되고 참조가 없으면 회원이 조용히 삭제된다. JPA 가 지우는 건 DB 행뿐이라 디스크 파일은 TransactionalFileRemover 가 커밋 후에 지우고 실패분은 OrphanFileCleanupScheduler 가 치운다 — DB 는 롤백되지만 파일 시스템은 롤백되지 않기 때문.
source: session
distilled: true
distilled_at: 2026-10-17
---

## 배운 개념

### 1. 거는 기준 — 생명주기 소유

> **부모가 자식의 생명주기를 단독으로 소유할 때만 건다.** 자식이 오직 그 부모 하나에만 속하고, 부모 없이는 존재할 의미가 없을 때.

| | 부모 없이 의미? | 다른 부모와 공유? | cascade |
|---|---|---|---|
| `Post → AttachedFile` | 없음 | 안 함 | ✅ |
| `Post → Member`(작성자) | 있음 (글보다 먼저 존재) | 여러 글·댓글이 공유 | ❌ |

- study-board 에서 cascade 가 걸린 곳은 `Post.files` 단 한 군데. 프로젝트 전체 `@OneToMany` 도 이것 하나.

### 2. `@ManyToOne` 에 cascade 를 걸면

```java
@ManyToOne(fetch = FetchType.LAZY, cascade = CascadeType.ALL)   // ❌
@JoinColumn(name = "author_id", nullable = false)
private Member author;
```

```
post (id=1, author_id=7)  ← 지금 지우는 글
post (id=2, author_id=7)  ← 같은 작성자의 다른 글
post (id=3, author_id=7)

delete from post where id = 1
delete from member where id = 7    ← cascade REMOVE 전이
```

- cascade 는 **선언한 방향으로만** 전이. Member 에는 posts 컬렉션도 cascade 도 없으므로 다른 글을 지워 줄 주체가 없다.
- 다른 글이 `author_id = 7` 을 참조 → **FK 제약조건 위반** → `DataIntegrityViolationException` → 트랜잭션 **롤백** → 글 1번도 안 지워짐.
- **더 무서운 경우**: 그 회원을 참조하는 행이 하나도 없으면(글 하나, 댓글·좋아요 없음) DELETE 가 **성공** → 글 하나 지웠는데 **회원 계정이 조용히 사라짐**. 에러가 없어 발견도 늦다.

### 3. JPA 가 지우는 건 DB 행뿐 — 디스크 파일은?

```java
// PostService.delete / deleteFile 첫 줄
fileRemover.removeAfterCommit(target.getStoredName());
```

1. 트랜잭션 안에서는 "지워도 된다"는 **예약만** (`TransactionSynchronization` 등록)
2. **커밋 확정 후** 디스크에서 실제 삭제
3. 커밋 후 삭제가 실패해 남은 파일은 `OrphanFileCleanupScheduler` 가 나중에 치움 (보험)
4. 트랜잭션 밖에서 불리면 미룰 곳이 없으므로 즉시 삭제

**왜 커밋 후인가** — DB 는 롤백되지만 파일 시스템은 롤백되지 않는다.

| 순서 | 실패 시 | 결과 |
|---|---|---|
| 디스크 먼저 → DB 나중 | DB 롤백 | DB 행은 살아 있는데 파일 없음 → 목록엔 보이고 다운로드 불가 (데이터 소실) |
| DB 커밋 → 디스크 나중 | 디스크 삭제 실패 | 안 쓰이는 파일이 남을 뿐 → 스케줄러가 치움 (감당 가능) |

→ **두 실패 중 감당할 수 있는 쪽을 고른 설계.** `@TransactionalEventListener(AFTER_COMMIT)` 와 같은 발상.

### 4. 면접 답변 (30초)

> "cascade 는 부모 엔티티에 한 persist·remove 같은 영속성 작업을 자식에게 전이하는 옵션이고, orphanRemoval 은 자식이 부모의 컬렉션에서 빠져 고아가 되면 그 자식을 삭제하는 옵션입니다. cascade 는 부모에게 일어난 일을, orphanRemoval 은 컬렉션에서 빠진 일을 보고 발동합니다.
> study-board 의 Post–AttachedFile 에 둘 다 건 이유는, 글을 저장할 때 파일도 함께 저장하려면 PERSIST 전이가 필요한데 이건 cascade 만 해 주고, 파일 하나만 삭제하는 기능은 컬렉션에서 빼는 방식이라 orphanRemoval 이 있어야 DB 행이 지워지기 때문입니다. 글 삭제는 두 옵션이 겹칩니다.
> 거는 기준은 부모가 자식의 생명주기를 단독으로 소유하느냐입니다. 파일은 글 없이 의미가 없어서 걸었고, 작성자 같은 @ManyToOne 에 걸면 글 삭제가 회원 삭제로 번지기 때문에 걸지 않았습니다.
> 그리고 JPA 가 지우는 건 DB 행뿐이라, 디스크 파일은 커밋 후에 지우고 실패분은 스케줄러가 치우도록 했습니다."

## 오답·헷갈린 점

- **전**: Member 에 cascade 를 걸면 "작성자가 지워짐에 따라 작성한 다른 글도 지워진다" → **후**: cascade 는 선언한 방향(Post → Member)으로만 전이. 다른 글은 FK 위반 → **전체 롤백**. 참조가 없으면 회원만 조용히 삭제.
- **전**: 디스크 파일은 `OrphanFileCleanupScheduler` 가 지운다 → **후**: 주 담당은 **커밋 후** `TransactionalFileRemover.removeAfterCommit`. 스케줄러는 실패분을 치우는 보험.
- ✅ 생명주기 소유 기준("post 는 file 의 생명주기를 소유하지만 member 는 소유하지 않는다")은 스스로 정확히 답함.

## Q&A

- **Q. Post.author 에 cascade=ALL 을 걸고 글을 삭제하면?** A. 회원 DELETE 전이 → 다른 글이 참조하면 FK 위반으로 롤백(글도 안 지워짐), 참조가 없으면 회원이 조용히 삭제.
- **Q. File 엔 안전하고 Member 엔 위험한 이유?** A. Post 는 File 의 생명주기를 소유하지만 Member 의 생명주기는 소유하지 않는다.
- **Q. 디스크 파일은 언제, 무엇이 지우나?** A. 커밋 후 TransactionalFileRemover. 실패분은 OrphanFileCleanupScheduler.
- **Q. 왜 커밋 후인가?** A. DB 는 롤백되지만 파일 시스템은 롤백되지 않음. 고아 파일은 치울 수 있지만 사라진 파일은 되돌릴 수 없다.

## 다음 학습

- `TransactionSynchronization.afterCommit` vs `@TransactionalEventListener(AFTER_COMMIT)` 구현 비교 → [[transactional-event-listener]]
- 회원 탈퇴 시 그 회원의 글·파일은 어떻게 처리하나 (soft delete? 익명화?)
- 연결: [[2026-10-16-cascade-vs-orphanremoval]], [[transactional-rollback-rules]]
