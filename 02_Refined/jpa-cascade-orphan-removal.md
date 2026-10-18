---
type: refined
slug: jpa-cascade-orphan-removal
tags: [cascade, 영속성전이, 영속성 전이, CascadeType, CascadeType.ALL, PERSIST, MERGE, REMOVE, REFRESH, DETACH, orphanRemoval, 고아객체, 고아 객체, 고아엔티티, orphan, OneToMany, ManyToOne, mappedBy, 생명주기, lifecycle, 단일소유, single-owner, 생명주기소유, FK제약조건, DataIntegrityViolationException, on-delete-cascade, DB-cascade, 첨부파일, 파일삭제, 디스크파일, JPA, Hibernate, 면접답변]
topic: cascade 와 orphanRemoval 의 발동 조건 차이 — cascade 는 부모에 한 영속성 작업을 전이하고 orphanRemoval 은 컬렉션에서 빠진 자식을 삭제한다. 둘 다 거는 이유와 거는 기준(생명주기 단독 소유)
summary: cascade(영속성 전이)는 부모 엔티티에 한 persist·remove·merge·detach·refresh 를 연관 엔티티에 똑같이 적용하는 JPA 기능이고, orphanRemoval 은 자식이 부모의 컬렉션에서 빠져 고아가 되면 그 자식을 DELETE 한다. 1:N 의 FK 는 자식에만 있어 mappedBy 컬렉션에서 빼는 것만으로는 SQL 이 나가지 않으므로 "자식 하나 삭제"에는 orphanRemoval 이 필요하다. orphanRemoval 은 부모 삭제 시에도 자식을 지우므로 삭제는 두 옵션이 겹치고, cascade 만 해 주는 건 PERSIST 다. 부모가 자식의 생명주기를 단독으로 소유할 때만 걸며, @ManyToOne 에 걸면 자식 삭제가 부모(회원) 삭제로 번진다.
contributors: [dongju]
source_refs:
  - https://jakarta.ee/specifications/persistence/3.1/
  - https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#pc-cascade
updated: 2026-10-17
---

# cascade 와 orphanRemoval

## 개념 정의

```java
@OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)
private List<AttachedFile> files = new ArrayList<>();
```

> **cascade(영속성 전이)** — 부모 엔티티에 하는 영속성 작업(persist, remove, merge, detach, refresh)을 연관 엔티티에도 똑같이 적용하는 **JPA 기능**.
>
> **orphanRemoval** — 자식이 부모의 **컬렉션에서 빠져** 고아가 되면 그 자식을 DELETE 한다.

| | 무엇을 보고 발동하나 |
|---|---|
| `cascade = REMOVE` | **부모가 삭제**될 때 자식도 삭제 |
| `orphanRemoval = true` | 자식이 **부모의 컬렉션에서 빠질** 때 삭제 |

- cascade 는 DB 의 `on delete cascade` 와 **별개**다. FK 에 DB cascade 가 없어도 동작한다.
- `CascadeType` 에 조회(R)는 없다. 수정(U)은 변경 감지가 하는 일이라 cascade 와 무관하다 ([[dirty-checking]]).
- `ALL` = PERSIST + MERGE + REMOVE + REFRESH + DETACH.

## 작동 원리

### 게시판 예시 — 세 장면

| 장면 | 코드 | 담당 |
|---|---|---|
| 글 작성 | `post.addFile(f); postRepository.save(post);` | cascade **PERSIST** — 파일 저장 코드 없이 INSERT |
| 글 수정 시 파일 추가 | 영속 `post` 에 `addFile` 만 (save 없음) | 플러시 때 **PERSIST** 전이 → INSERT |
| 글 삭제 | `postRepository.delete(post)` | cascade **REMOVE** (orphanRemoval 과 겹침) |
| 파일 하나 삭제 | `post.getFiles().remove(target)` | **orphanRemoval** |

cascade 가 없으면 글 작성 시 `AttachedFile` 은 **비영속** 그대로 → INSERT 안 나감. 디스크에 파일을 먼저 썼다면 DB 행 없는 **고아 파일**이 된다.

### 1:N 의 FK 는 자식에만 있다

```
post (id, title, author_id)          attached_file (id, stored_name, post_id) ← FK 는 여기에만
```

- `post.files` 리스트는 자바 참조일 뿐이고, DB 에서 관계를 기록하는 곳은 `attached_file.post_id` 하나다 ([[jpa-association-owner]]).

### orphanRemoval 이 없으면 — "보이는데 다운로드가 안 되는 파일"

`cascade = ALL` 만 있고 `post.getFiles().remove(target)` 를 하면:

```
→ 자바 메모리: post.files 에서 target 이 빠짐
→ target.post: 여전히 post 를 가리킴 (관계의 주인은 안 바뀜)
→ 부모에 remove 호출 없음 → 전이할 작업 없음
→ 나가는 SQL: 없음
```

```
[요청 1] 리스트에서만 뺌 → 트랜잭션 종료 → 그 post 객체도 사라짐 (디스크 파일은 따로 삭제됨)
[요청 2] 새로고침 → 새 영속성 컨텍스트가 DB 에서 files 를 다시 읽음
       → 지운 파일이 목록에 다시 보임 → 클릭하면 디스크에 없어 다운로드 실패
```

→ orphanRemoval 은 편의가 아니라 **정확성**을 위한 옵션이다.

### 왜 둘 다 거나

`cascade` 를 빼고 `orphanRemoval` 만 남기면:

| 장면 | 결과 | 이유 |
|---|---|---|
| 글 작성 | 💥 깨짐 | PERSIST 전이 없음 |
| 글 삭제 | ✅ | 부모를 잃은 자식도 고아 → 함께 삭제 (JPA 명세: orphanRemoval 이면 cascade=REMOVE 불필요) |
| 파일 하나 삭제 | ✅ | orphanRemoval 본래 역할 |

> **삭제는 두 옵션이 겹친다. cascade 만 해 주는 건 PERSIST(등록).**

## 트레이드오프 / 한계

### 거는 기준 — 부모가 자식의 생명주기를 단독 소유할 때

| | 부모 없이 의미? | 다른 부모와 공유? | cascade |
|---|---|---|---|
| `Post → AttachedFile` | 없음 | 안 함 | ✅ |
| `Post → Member`(작성자) | 있음 | 여러 글·댓글이 공유 | ❌ |

`@ManyToOne` 에 걸면:

```java
@ManyToOne(fetch = FetchType.LAZY, cascade = CascadeType.ALL)   // ❌
@JoinColumn(name = "author_id", nullable = false)
private Member author;
```

- 글 삭제 → `delete from member where id = 7` 전이.
- cascade 는 **선언한 방향으로만** 전이된다. 같은 작성자의 다른 글은 지워 줄 주체가 없다 → 다른 글이 `author_id = 7` 을 참조 → **FK 제약조건 위반 → 전체 롤백** (글 삭제도 실패).
- 더 무서운 경우: 그 회원을 참조하는 행이 하나도 없으면 DELETE 가 **성공** → 글 하나 지웠는데 **회원 계정이 조용히 사라진다**.

### 기타

- cascade REMOVE 는 자식을 **하나씩** DELETE 한다(자식 10개 → DELETE 10 + 1). 대량이면 DB `on delete cascade` 나 벌크 삭제를 검토.
- orphanRemoval 이 걸린 자식을 다른 부모 컬렉션으로 옮기면 원래 부모 쪽에서는 고아로 판정될 수 있다 — 자식을 부모 간에 옮기는 도메인이면 걸지 않는다.
- JPA 가 지우는 건 **DB 행뿐**이다. 디스크 파일 같은 외부 자원은 별도로, **커밋 후에** 지워야 한다(DB 는 롤백되지만 파일 시스템은 롤백되지 않는다) — [[transactional-event-listener]].

> **면접 답변** — "cascade는 부모 엔티티에 한 persist·remove 같은 영속성 작업을 자식에게 전이하는 옵션이고, orphanRemoval은 자식이 부모의 컬렉션에서 빠져 고아가 되면 그 자식을 삭제하는 옵션입니다. 글을 저장할 때 파일도 함께 저장하려면 PERSIST 전이가 필요한데 이건 cascade만 해 주고, 파일 하나만 삭제하는 기능은 컬렉션에서 빼는 방식이라 orphanRemoval이 있어야 DB 행이 지워집니다. 글 삭제는 두 옵션이 겹칩니다. 거는 기준은 부모가 자식의 생명주기를 단독으로 소유하느냐이고, 작성자 같은 @ManyToOne에 걸면 글 삭제가 회원 삭제로 번지기 때문에 걸지 않습니다."

> [!WARNING]
> **오답 코너**
> - **"cascade 는 조인된 테이블이 CRUD 될 때 자식도 CRUD"** — 테이블이 아니라 **엔티티**에 대한 **영속성 작업**의 전이. 조회는 대상이 아니고 수정은 변경 감지.
> - **"cascade 가 없으면 post 에 FK 가 등록되고 file 만 안 된다"** — post 에는 FK 가 없다. 1:N 의 FK 는 자식에만.
> - **"cascade=ALL 만 있어도 컬렉션에서 빼면 행이 지워진다"** — 부모에 remove 가 없어 전이할 작업이 없고, 관계의 주인이 아닌 리스트 변경이라 **SQL 0건**.
> - **"SQL 은 없지만 새로고침하면 안 보인다"** — 다음 요청은 DB 를 새로 읽으므로 **보인다**(그리고 디스크 파일이 지워졌다면 다운로드 불가).
> - **"orphanRemoval 만 있으면 글 삭제가 깨진다"** — 부모를 잃은 자식도 고아라 함께 삭제된다. 깨지는 건 **글 작성**(PERSIST).
> - **"Member 에 cascade 를 걸면 작성자의 다른 글도 지워진다"** — 선언한 방향으로만 전이. 다른 글은 FK 위반 → **롤백**.

## 복습 체크

- [ ] cascade 와 orphanRemoval 이 각각 **무엇을 보고** 발동하는지 한 줄씩 말할 수 있는가?
- [ ] `CascadeType` 다섯 가지를 대고, 조회·수정이 없는 이유를 설명할 수 있는가?
- [ ] `save(post)` / `delete(post)` 에 전이되는 타입은?
- [ ] cascade=ALL 만 있고 컬렉션에서 자식을 빼면 SQL 이 몇 개 나가며, 새로고침하면 어떻게 보이는가?
- [ ] orphanRemoval 만 남기면 어느 장면이 깨지는가? 글 삭제는 왜 괜찮은가?
- [ ] 두 옵션을 거는 기준을 "생명주기"로 말할 수 있는가?
- [ ] `@ManyToOne` 에 cascade 를 걸면 생기는 두 가지 결말은?
- [ ] JPA 가 지우지 못하는 것은 무엇이고, 언제 지워야 하는가?

## 관련

[[jpa-embedded-value-type]] · [[jpa-association-owner]] · [[jpa-entity-lifecycle-merge]] · [[persistence-context]] · [[dirty-checking]] · [[transactional-event-listener]] · [[transactional-rollback-rules]]
