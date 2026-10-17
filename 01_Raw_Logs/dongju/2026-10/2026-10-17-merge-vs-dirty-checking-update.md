---
date: 2026-10-17
type: raw
author: dongju
tags: [merge, 변경 감지, dirty checking, 수정 방식, update, 전체 필드 덮어쓰기, null 덮어쓰기, Spring Data save, 폼 바인딩, 소유자 검증, ownership check, 권한 검증, findOwned, cascade MERGE, orphanRemoval, not null 제약, viewCount, likeCount, 엔티티 메서드, 도메인 메서드, PostService.update, study-board, 면접 답변]
topic: 수정에 merge(=id 있는 객체 save) 대신 조회 + 변경 감지를 쓰는 이유 — merge 는 모든 필드를 덮어써 폼에 없던 값이 날아가고, id 만 맞으면 남의 글도 덮어쓴다
summary: 폼 값으로 new Post(id, ...) 를 만들어 save() 하면 merge 가 되어 폼에 없던 viewCount·likeCount 가 0, author 가 null(not null 제약 위반으로 롤백), files 가 빈 리스트(cascade MERGE + orphanRemoval 로 첨부파일 삭제 가능)로 덮어써지고, id 만 바꿔 보내면 남의 글도 수정된다. study-board 는 findOwned(조회 + 소유자 검증) → post.update(title, content, category)(필요한 필드만) → 변경 감지로 수정하며, 프로젝트 전체에 merge 직접 호출은 없고 save() 는 새 엔티티에만 쓴다.
source: session
distilled: false
---

## 배운 개념

### 1. study-board 의 수정 방식

```java
// PostService.update — 실제 코드
Post post = findOwned(id, memberId);                                     // 조회 + 소유자 검증
post.update(form.getTitle(), form.getContent(), form.getCategory());     // 바꿀 필드만
// 커밋 시 변경 감지 → UPDATE
```

- 프로젝트 전체에 `em.merge` 직접 호출 **0건**, `save()` 는 전부 **새 엔티티**에만.

### 2. 만약 merge 방식으로 바꾸면

```java
// 가상의 코드
@Transactional
public void update(Long id, PostForm form) {
    Post post = new Post(id, form.getTitle(), form.getContent(), form.getCategory());
    // author = null, viewCount = 0, likeCount = 0, createdAt = null, files = 빈 리스트
    postRepository.save(post);   // id 가 있으니 merge
}
```

merge 는 인자의 **모든 필드**를 영속 엔티티에 덮어쓴다.

| 필드 | 폼에서 만든 객체 | merge 후 |
|---|---|---|
| `viewCount` / `likeCount` | 0 | 1,000 → **0**, 50 → **0** |
| `author` | null | 작성자 → **null** |
| `files` | 빈 리스트 | 파일 목록 → **빈 리스트** |

- `author_id` 는 `nullable = false` → 플러시 때 **제약조건 위반 → 롤백**. 데이터는 지켜지지만 수정 기능이 동작하지 않는다.
- 제약이 없었다면 **조용히** 숫자 0, 작성자 소실, 그리고 `cascade = ALL`(MERGE 포함) + `orphanRemoval` 로 **첨부파일 행까지 삭제**될 수 있다 (10/16 cascade·orphanRemoval 과 연결).
- **보안**: merge 는 id 만 맞으면 그 행을 덮어쓴다 → 폼의 숨은 `id` 를 남의 글 번호로 바꿔 보내면 **남의 글이 수정**된다.

### 3. 조회 + 변경 감지가 안전한 이유

> ① 엔티티 메서드(`post.update(...)`)가 **바꿀 필드만** 바꾼다.
> ② **조회하는 순간** 소유자 같은 규칙을 검증할 수 있다 (`findOwned` 가 남의 글이면 수정 전에 예외).

- 프로젝트 크기나 id 생성 전략과는 **무관**한 선택이다.

### 4. 면접 답변 (30초)

> "준영속 엔티티는 한때 영속이었고 id도 있지만, 영속성 컨텍스트가 더 이상 관리하지 않는 상태입니다. 트랜잭션이 끝나 영속성 컨텍스트가 닫히거나 detach·clear를 하면 그렇게 되고, 그 뒤의 변경은 변경 감지가 안 돼서 DB에 반영되지 않습니다.
> persist는 비영속 엔티티를 영속으로 등록해서 커밋 때 INSERT하는 메서드라, 준영속에 쓰면 예외가 납니다. merge는 준영속 엔티티의 id로 1차 캐시나 DB에서 영속 엔티티를 찾고, 거기에 모든 필드를 덮어쓴 뒤 그 영속 객체를 반환합니다. 넘긴 객체는 준영속 그대로예요. Spring Data의 save()도 id가 있으면 merge를 호출합니다.
> study-board는 수정에 merge를 쓰지 않고, findOwned로 조회한 뒤 엔티티 메서드로 필요한 필드만 바꿔 변경 감지에 맡깁니다. merge는 폼에 없는 필드까지 덮어써서 조회수나 작성자가 날아갈 수 있고, 조회하는 단계에서 소유자 검증도 함께 할 수 있기 때문입니다."

## 오답·헷갈린 점

- **전**: merge 방식의 문제 = "값이 사라져서 0이 된다" → **후**: 숫자뿐 아니라 `author` null(제약 위반), `files` 비움(orphanRemoval 로 파일 삭제 가능), **남의 글 수정**까지.
- **전**: 조회 + 변경 감지가 안전한 이유 = "변경한 값만 변경된다" (1개만) → **후**: + **조회 시점의 소유자 검증**(`findOwned`).
- **전**: merge 를 안 쓰는 이유 = "프로젝트가 작기 때문에" (18:42) → "아이덴티티 때문이다" (18:45) → **후**: ① 바꿀 필드만 바꾼다, ② 조회하면서 소유자를 검증한다. 크기·id 전략과 무관.
  - ⚠️ **본인 말로 다시 말하지 못하고 인터뷰 종료** — 다음 복습 때 이 질문부터 다시 확인할 것.
  - ⚠️ 같은 날 앞선 IDENTITY 인터뷰 내용과 섞임 — 긴 세션 끝에 주제 간 간섭 발생.

## Q&A

- **Q. 폼 값으로 만든 Post 를 save() 하면?** A. id 가 있어 merge → 모든 필드 덮어쓰기. 조회수·좋아요 0, author null(제약 위반 롤백), files 비움.
- **Q. merge 의 보안 문제?** A. id 만 맞으면 남의 글도 덮어쓴다.
- **Q. study-board 가 조회 + 변경 감지로 수정하는 이유?** A. ① 바꿀 필드만, ② 조회 시 소유자 검증.

## 다음 학습

- 🔁 **복습 1순위**: "study-board 가 merge 대신 조회 + 변경 감지로 수정하는 이유 두 가지"를 본인 말로
- 엔티티 대신 DTO 를 서비스 경계로 주고받는 설계가 merge 문제를 어떻게 원천 차단하나
- 준영속 엔티티를 다른 트랜잭션에서 쓸 때 `LazyInitializationException` 과의 관계 → [[lazy-loading-proxy]]
- 연결: [[2026-10-17-entity-lifecycle-persist-merge]], [[dirty-checking]], [[2026-10-16-cascade-vs-orphanremoval]]
