---
date: 2026-10-17
type: raw
author: dongju
tags: [연관관계 편의 메서드, convenience method, addFile, setPost, 양방향 연관관계, bidirectional, 연관관계의 주인, owning side, mappedBy, cascade, PERSIST, 영속성 전이, 컬렉션, collection, 비영속, transient, FK null, 주인 없는 행, 고아 행, 메모리 정합성, validateFileCount, study-board, 면접 답변]
topic: 주인만 보면 된다면서 왜 양쪽을 다 세팅하나 — 반대쪽만 세팅하면 post_id NULL 행, 주인만 세팅하면 cascade 통로가 비어 아예 저장 안 됨. 그래서 연관관계 편의 메서드
summary: FK 는 주인만 보고 쓰지만 비주인 컬렉션도 쓸모가 있다. study-board 의 create() 는 cascade PERSIST 로 파일을 저장하는데 cascade 는 Post.files 컬렉션을 따라 전이되므로, files.add 를 빼면(주인만 세팅) 파일이 비영속으로 남아 INSERT 가 없고, file.setPost 를 빼면(반대쪽만 세팅) INSERT 는 되지만 post_id 가 NULL 인 주인 없는 행이 생긴다. 같은 트랜잭션의 메모리 상태(post.getFiles().size())도 비주인 쪽이 책임지므로 addFile 에서 양쪽을 함께 세팅한다.
source: session
distilled: false
---

## 배운 개념

### 1. 편의 메서드

```java
// Post.java
public void addFile(AttachedFile file) {
    files.add(file);       // ① 주인이 아닌 쪽 — cascade 통로 + 메모리 정합성
    file.setPost(this);    // ② 주인 — FK(post_id) 값
}
```

```java
// PostService.create()
List<AttachedFile> files = fileStore.storeFiles(form.getFiles());
files.forEach(post::addFile);
return postRepository.save(post).getId();     // 파일 저장은 cascade PERSIST 에 의존
```

### 2. ② `setPost` 를 빼면 (반대쪽만 세팅)

- `files.add(file)` 는 있으므로 `save(post)` → cascade 가 `post.files` 를 열어 `file` 을 만남 → **INSERT 됨**.
- 주인 필드 `file.post` 는 null → JPA 는 주인만 보고 FK 를 쓰므로:

```
attached_file ( id=10, stored_name='a1b2.png', original_name='공부계획.png', post_id=NULL )
```

- 다른 컬럼은 다 채워진 채 **`post_id` 만 NULL** — 어느 글에도 속하지 않는 행.
- 다음 요청에서 상세를 열면 `where post_id = ?` 에 안 걸려 **파일이 안 보임**. 디스크엔 파일, DB 엔 행이 있는데 고아. 에러 없음.
- study-board 의 `attached_file.post_id` 는 `not null` 이 아니라 DB 가 막아 주지 못한다.

### 3. ① `files.add` 를 빼면 (주인만 세팅)

cascade 는 `@OneToMany(... cascade = ALL)` 가 붙은 **`files` 필드를 따라가는 길**이다.

```
save(post)
  → post 를 persist
  → cascade 가 post.files 를 연다 → 리스트가 비어 있다
  → 전이할 대상: 없음
file 의 상태: 비영속  (file.post 에는 post 가 들어 있지만, 아무도 file 을 persist 하지 않았다)
→ attached_file INSERT 없음
```

- "비주인은 무시된다"는 말은 **FK 를 쓸 때만** 맞다. 비주인 컬렉션은 **cascade 의 통로**다.
- 단, 이건 cascade 로 자식을 저장하는 study-board 구조에서의 결과. `fileRepository.save(file)` 로 직접 저장하면 주인만 세팅해도 DB 는 맞다 — 대신 그 트랜잭션 안에서 `post.getFiles()` 가 실제와 달라진다.

### 4. 정리

| 세팅한 쪽 | 결과 |
|---|---|
| 반대쪽만 (`files.add`) | cascade 로 INSERT 는 되지만 `post_id = NULL` (주인 없는 행) |
| 주인만 (`setPost`) | cascade 통로가 비어 **비영속 → INSERT 없음** |
| 양쪽 다 (`addFile`) | ✅ |

- 비주인 쪽은 같은 트랜잭션 안의 **메모리 상태**도 책임진다. study-board 의 `update()` 는 `validateFileCount(post.getFiles().size(), form)` 로 리스트 크기를 쓴다.

### 5. 면접 답변 (30초)

> "양방향 연관관계에서 객체는 서로를 가리키는 참조가 두 개인데, 테이블은 FK가 하나뿐입니다. 두 참조 값이 다를 때 FK에 무엇을 쓸지 정하려면 둘 중 하나를 기준으로 삼아야 하고, 그게 연관관계의 주인입니다. 주인만 FK를 등록·수정하고, 주인이 아닌 쪽은 조회만 합니다. 보통 FK가 있는 N 쪽이 주인이 되며, study-board에서는 post_id를 가진 AttachedFile.post가 주인입니다.
> mappedBy = "post"는 '나는 주인이 아니고, AttachedFile의 post 필드에 의해 매핑된다'는 선언입니다.
> 그렇다고 한쪽만 세팅하면 안 돼서 addFile 편의 메서드에서 양쪽을 함께 세팅합니다. 반대쪽 리스트만 넣으면 cascade로 저장은 되지만 post_id가 null인 행이 생기고, 주인만 세팅하면 cascade가 컬렉션을 따라 전이되기 때문에 파일이 아예 저장되지 않습니다."

## 오답·헷갈린 점

- **전**: `setPost` 를 빼면 "행이 생기지 않는다, null 값이 들어간다" (자기모순) → **후**: 행은 **생기고**, `post_id` 만 NULL.
- **전**: 그 행은 "빈 엔티티" → **후**: stored_name 등 다른 값은 다 채워진 **주인 없는 행**.
- **전**: `files.add` 를 빼도 "영속성의 전이로 저장된다" → **후**: cascade 는 **컬렉션을 따라** 전이되는데 컬렉션이 비어 있어 전이 대상 없음 → INSERT 없음.
- **전**: 그때 file 의 상태는 "null" → **후**: **비영속** (객체는 존재하지만 영속성 컨텍스트가 모름).
- **전**: 최종 답변에서 "주인만 세팅 → 가비지 엔티티 / 반대쪽만 세팅 → 비영속" → **후**: **반대**. 주인만 → 비영속(INSERT 없음), 반대쪽만 → post_id NULL 행. 결과는 이해했는데 주어가 뒤바뀜. ⚠️ 면접 답변으로 그대로 말하면 치명적 — 표를 외울 것.

## Q&A

- **Q. `addFile` 에서 `setPost` 를 빼면?** A. INSERT 는 되지만 post_id NULL 인 주인 없는 행. 다음 요청에 안 보임.
- **Q. `files.add` 를 빼면?** A. cascade 통로가 비어 파일이 비영속 → INSERT 없음.
- **Q. "비주인은 무시된다"는데 왜 세팅하나?** A. FK 쓸 때만 무시. 비주인 컬렉션은 cascade 통로이고 같은 트랜잭션의 메모리 상태를 책임진다.

## 다음 학습

- 편의 메서드를 양쪽 엔티티에 다 두면 서로 호출하며 생기는 무한 루프, 한쪽에만 두는 관례
- `AttachedFile.post` 의 `@JoinColumn` 에 `nullable = false` + 마이그레이션으로 `post_id not null` → 오늘의 "주인 없는 행" 버그를 DB 가 막아 주는지 (study-board 개선 후보)
- 연결: [[2026-10-17-association-owner-mappedby]], [[2026-10-16-cascade-vs-orphanremoval]]
