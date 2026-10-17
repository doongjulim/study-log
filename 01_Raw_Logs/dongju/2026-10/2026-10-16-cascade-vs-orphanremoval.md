---
date: 2026-10-16
type: raw
author: dongju
tags: [cascade, 영속성 전이, CascadeType, CascadeType.ALL, PERSIST, MERGE, REMOVE, REFRESH, DETACH, orphanRemoval, 고아 객체, orphan, 고아 엔티티, OneToMany, mappedBy, 연관관계 주인, owning side, 외래 키, FK, 1:N, 비영속, transient, 영속, 변경 감지, dirty checking, 플러시, 연관관계 편의 메서드, addFile, setPost, Post, AttachedFile, study-board]
topic: cascade 와 orphanRemoval 은 발동 조건이 다르다 — cascade 는 부모에 한 영속성 작업을 전이하고, orphanRemoval 은 컬렉션에서 빠진 자식을 삭제한다. Post-AttachedFile 에 둘 다 건 이유
summary: cascade(영속성 전이)는 부모 엔티티에 한 persist·remove·merge·detach·refresh 를 연관 엔티티에 똑같이 적용하는 JPA 기능이고, orphanRemoval 은 자식이 부모의 컬렉션에서 빠지면 그 자식을 DELETE 한다. 1:N 의 FK 는 자식(attached_file.post_id)에만 있어 mappedBy 컬렉션에서 빼는 것만으로는 SQL 이 나가지 않으므로 파일 하나 삭제에는 orphanRemoval 이 필요하다. orphanRemoval 은 부모 삭제 시에도 자식을 지우므로 삭제는 두 옵션이 겹치고, cascade 만 해 주는 건 PERSIST(글 저장 시 파일 함께 저장)다.
source: session
distilled: true
distilled_at: 2026-10-17
---

## 배운 개념

### 1. 매핑

```java
// Post.java
@OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)
private List<AttachedFile> files = new ArrayList<>();

public void addFile(AttachedFile file) {
    files.add(file);
    file.setPost(this);      // 주인 쪽(AttachedFile.post = post_id FK)까지 세팅해야 DB 에 관계가 남는다
}
```

### 2. cascade — 영속성 전이

> **cascade 는 부모 엔티티에 하는 영속성 작업(persist, remove, merge, detach, refresh)을 연관 엔티티에도 똑같이 적용하는 JPA 기능.**

- DB 의 `on delete cascade` 와 별개. study-board 의 `attached_file` FK 에는 DB cascade 가 **없다**.
- `CascadeType` 에 조회(R)는 없다. 수정(U)은 변경 감지가 하는 일이라 cascade 와 무관.
- `ALL` = PERSIST + MERGE + REMOVE + REFRESH + DETACH.

| 코드 | 전이되는 타입 |
|---|---|
| `postRepository.save(post)` (create) | `PERSIST` |
| `postRepository.delete(post)` (delete) | `REMOVE` |
| 영속 `post` 에 `addFile` 만 (update, save 없음) | 플러시 때 `PERSIST` 전이 → INSERT |

cascade 가 없으면 `create()` 에서 `AttachedFile` 은 **비영속** 그대로 → `attached_file` INSERT 안 나감.
그런데 `fileStore.storeFiles()` 가 이미 디스크에 파일을 썼으므로 → DB 행 없는 **고아 파일**.

### 3. 1:N 의 FK 는 자식 쪽에만 있다

```
post                          attached_file
┌────┬───────┬───────────┐    ┌────┬──────────────┬─────────┐
│ id │ title │ author_id │    │ id │ stored_name  │ post_id │ ← FK 는 여기에만
└────┴───────┴───────────┘    └────┴──────────────┴─────────┘
```

- `post` 에는 파일 관련 컬럼이 없다 (한 컬럼에 여러 값을 넣을 수 없으니 FK 는 항상 N 쪽).
- `post.files` 리스트는 **자바 객체 참조**일 뿐. DB 에서 관계를 기록하는 곳은 `attached_file.post_id` 하나.
- `mappedBy = "post"` = "이 리스트는 주인이 아니고, 진짜 관계는 `AttachedFile.post` 가 관리한다."

### 4. orphanRemoval

> **orphanRemoval 은 자식이 부모의 컬렉션에서 빠져 고아가 되면 그 자식을 DELETE 한다.**

```java
@Transactional
public void deleteFile(Long postId, Long fileId, Long memberId) {
    Post post = findOwned(postId, memberId);
    AttachedFile target = ...;
    fileRemover.removeAfterCommit(target.getStoredName());
    post.getFiles().remove(target);   // orphanRemoval 로 DB 에서도 삭제 (1행)
}
```

**orphanRemoval 없이 `cascade = ALL` 만 있다면:**

```
post.getFiles().remove(target)
  → 자바 메모리: post.files 에서 target 이 빠짐
  → target.post: 여전히 post 를 가리킴 (주인 쪽 안 바뀜)
  → 부모에 remove 호출 없음 → 전이할 작업 없음
  → 나가는 SQL: 없음
```

```
[요청 1] deleteFile() → 리스트에서만 뺌 → 트랜잭션 종료 → 영속성 컨텍스트와 함께 사라짐
                       → 커밋 후 디스크 파일은 removeAfterCommit 이 삭제
[요청 2] 새로고침 → 새 영속성 컨텍스트 → @EntityGraph 로 DB 에서 files 를 새로 읽음
                 → 지운 파일이 목록에 다시 보임 → 클릭하면 디스크에 없어서 다운로드 불가
```

→ orphanRemoval 은 편의가 아니라 **정확성**을 위한 옵션. 없으면 "DB 엔 있는데 파일은 없는" 상태가 생긴다.

### 5. 차이 — 발동 조건

| | 무엇을 보고 발동하나 |
|---|---|
| `cascade = REMOVE` | **부모가 삭제**될 때 자식도 삭제 |
| `orphanRemoval = true` | 자식이 **부모의 컬렉션에서 빠져** 고아가 될 때 삭제 |

### 6. 왜 둘 다 걸었나

`cascade` 를 빼고 `orphanRemoval` 만 남기면:

| 장면 | 결과 | 이유 |
|---|---|---|
| 글 작성 | 💥 깨짐 | `PERSIST` 전이 없음 → 파일 비영속 |
| 글 삭제 | ✅ 동작 | 부모를 잃은 자식도 고아 → 함께 삭제 (JPA 명세: orphanRemoval 이면 cascade=REMOVE 불필요) |
| 파일 하나 삭제 | ✅ 동작 | orphanRemoval 본래 역할 |

- **삭제는 두 옵션이 겹친다. cascade 만 해 주는 건 PERSIST(등록).**
- 그래서 `cascade = ALL` 로 글 저장 시 파일도 함께 저장하고, `orphanRemoval` 로 컬렉션에서 뺀 파일을 지운다.

## 오답·헷갈린 점

- **전**: cascade = "조인되어 있는 **테이블**이 부모가 **CRUD** 될 때 자식도 CRUD" → **후**: **엔티티**에 대한 **영속성 작업**의 전이. 조회는 전이 대상이 아니고, 수정은 변경 감지.
- **전**: cascade 정의 = "부모에 등록·삭제하면 전이" → **후**: MERGE·DETACH·REFRESH 까지. ALL 은 다섯 가지 전부.
- **전**: cascade 가 없으면 "post 엔티티에 FK 가 등록되고 file 엔티티는 등록이 안 되는 것?" → **후**: post 에는 FK 가 없다. 1:N 의 FK 는 자식(`attached_file.post_id`)에만. post 행은 INSERT 되고 attached_file 행은 안 생기며, `post.files` 는 메모리 속 참조일 뿐.
- **전**: `cascade = ALL` 만 있을 때 `files.remove(target)` → "테이블의 행은 다 지워진다" → **후**: 부모에 remove 가 없어 전이할 작업이 없고, 주인 아닌 쪽 리스트 변경이라 **SQL 없음**. (orphanRemoval 이 있어도 지워지는 건 **1행**)
- **전**: SQL 은 없는데 새로고침하면 "안 보인다" (자기모순) → **후**: 다음 요청은 새 영속성 컨텍스트로 DB 를 새로 읽음 → **보인다**. 디스크 파일은 지워져 **다운로드 불가**.
- **전**: orphanRemoval 만 있으면 글 삭제가 "cascade 부재로 깨진다" → **후**: 부모가 삭제된 자식도 고아 → **함께 삭제됨**. 깨지는 건 **글 작성**(PERSIST).
- ⚠️ 반복 테마: "다 지워진다", "CRUD 된다" 같은 뭉뚱그린 표현 → 어떤 작업이 어디로 전이되고 어떤 SQL 이 몇 개 나가는지로 환원 필요.
- ⚠️ 반복 테마: 객체 참조(리스트)와 DB 상태(FK)를 같은 것으로 취급 — 10/13 프록시 "조인" 혼동과 같은 계열(객체 레벨 vs DB 레벨).

## Q&A

- **Q. `save(post)` 만 했는데 attached_file INSERT 가 나가는 이유?** A. cascade PERSIST 전이.
- **Q. cascade 가 없으면?** A. 파일은 비영속 그대로, INSERT 없음. 디스크에는 이미 저장돼 고아 파일.
- **Q. save / delete 에 대응하는 CascadeType?** A. PERSIST / REMOVE.
- **Q. cascade=ALL 만 있고 `files.remove(target)` 이면 SQL?** A. 없음. 새로고침하면 다시 보이고 다운로드는 실패.
- **Q. 두 옵션의 차이?** A. cascade 는 부모에 일어난 일, orphanRemoval 은 컬렉션에서 빠진 일을 보고 발동.
- **Q. orphanRemoval 만 남기면 어디가 깨지나?** A. 글 작성만. 글 삭제·파일 삭제는 동작.

## 다음 학습

- cascade REMOVE 는 자식을 하나씩 DELETE (파일 10개 → DELETE 10 + 1). DB `on delete cascade`(study-board `comments` FK) 와 비교
- orphanRemoval 이 걸린 자식을 다른 부모 컬렉션으로 옮기면 어떻게 되나
- `addFile` 에서 `file.setPost(this)` 를 빼먹으면 `post_id` 에 무엇이 들어가나 (연관관계 편의 메서드)
- `show-sql` 로 deleteFile 실행 시 DELETE 1건 확인
- 연결: [[persistence-context]], [[dirty-checking]], [[2026-10-16-cascade-ownership-and-file-deletion]]
