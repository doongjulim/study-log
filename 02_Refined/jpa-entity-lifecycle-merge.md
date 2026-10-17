---
type: refined
slug: jpa-entity-lifecycle-merge
tags: [엔티티생명주기, 엔티티 생명주기, entity-lifecycle, 엔티티상태, 비영속, transient, new, 영속, managed, 준영속, detached, 삭제, removed, persist, merge, detach, clear, remove, PersistentObjectException, detached-entity-passed-to-persist, InvalidDataAccessApiUsageException, SimpleJpaRepository, save, isNew, Persistable, 전체필드덮어쓰기, null덮어쓰기, 변경감지, dirty-checking, 수정방식, 소유자검증, ownership-check, 폼바인딩, 엔티티메서드, JPA, Hibernate, Spring-Data-JPA, 면접답변]
topic: 엔티티의 네 가지 상태와 persist·merge 의 차이 — persist 는 비영속 객체 자체를 영속으로(준영속이면 예외), merge 는 id 로 얻은 영속 엔티티에 모든 필드를 덮어써 다른 객체를 반환. 그래서 수정은 merge(=id 있는 save) 대신 조회 + 변경 감지로
summary: 엔티티는 비영속 → persist → 영속 → 트랜잭션 종료·detach·clear → 준영속, 영속에서 remove → 삭제 상태가 된다. 준영속은 id 는 있지만 변경 감지·지연 로딩을 받지 못한다. persist 는 비영속을 새로 등록하며 준영속에 쓰면 예외가 난다. merge 는 인자의 id 로 1차 캐시나 DB 에서 영속 엔티티를 얻어 모든 필드를 덮어쓴 뒤 그 영속 객체를 반환하고, 인자는 준영속 그대로다. Spring Data save() 는 id 가 있으면 merge 를 호출하므로, 폼 값으로 만든 객체를 save 하면 폼에 없던 필드가 null·0 으로 덮이고 id 만 바꿔 보내면 남의 행도 수정된다. 수정은 조회(+소유자 검증) → 엔티티 메서드로 필요한 필드만 변경 → 변경 감지가 안전하다.
contributors: [dongju]
source_refs:
  - https://jakarta.ee/specifications/persistence/3.1/
  - https://docs.spring.io/spring-data/jpa/reference/jpa/entity-persistence.html
updated: 2026-10-17
---

# 엔티티 생명주기와 persist / merge

## 개념 정의

| 상태 | 들어가는 계기 |
|---|---|
| 비영속 (new/transient) | `new` — 영속성 컨텍스트와 무관한 자바 객체 |
| 영속 (managed) | `persist`, `find`/JPQL 조회, `merge` 의 **반환값** |
| 준영속 (detached) | 트랜잭션 종료(영속성 컨텍스트 종료), `em.detach(entity)`, `em.clear()` |
| 삭제 (removed) | 영속 엔티티에 `em.remove()` → 커밋 때 DELETE |

```java
Post post = new Post("제목", ...);      // 비영속

@Transactional
void save() {
    em.persist(post);                   // 영속
    post.setTitle("새 제목");            // → 커밋 때 스냅샷과 비교해 UPDATE
}                                       // 트랜잭션 종료 (OSIV off)

post.setTitle("또 바꿈");                // 준영속 — 무시됨. 필드 변경으로 상태가 바뀌지도 않는다
```

> **준영속** — 한때 영속이었고 **id 도 있지만**, 영속성 컨텍스트가 관리하지 않아 **변경 감지·지연 로딩을 받지 못하는** 상태 ([[dirty-checking]], [[lazy-loading-proxy]]).

## 작동 원리

### persist — 비영속 객체 자체를 등록

- 비영속 엔티티를 **그 객체 그대로** 영속으로 등록하고 커밋 때 INSERT.
- 준영속(id 가 채워진 엔티티)에 쓰면 → `PersistentObjectException: detached entity passed to persist` (스프링에선 `InvalidDataAccessApiUsageException`). 이미 있는 행을 또 INSERT 할 수 없으니 Hibernate 가 미리 막는다.

| 상황 | 결과 |
|---|---|
| 준영속에 `setTitle(...)` | **무시** |
| 준영속을 `persist(...)` | **예외** |

### merge — 다른 객체를 돌려준다

```java
@Transactional
void update(Post detached) {               // id=1
    Post merged = em.merge(detached);
    System.out.println(detached == merged); // false
    detached.setTitle("A");                 // 무시 (여전히 준영속)
    merged.setTitle("B");                   // 변경 감지 → title = "B"
}
```

```
em.merge(detached)
  → id 로 1차 캐시 조회 → 없으면 select ... where id = ?   ← 영속 엔티티를 얻는다
  → detached 의 "모든" 필드 값을 그 영속 엔티티에 덮어쓴다
  → 영속 엔티티를 반환. 인자는 준영속 그대로
```

- 흔한 실수: `em.merge(detached);` 의 **반환값을 버리고** 계속 `detached` 를 수정 → 이후 수정은 전부 사라진다.

### Spring Data `save()` 의 정체

```java
// SimpleJpaRepository.save()
if (entityInformation.isNew(entity)) {   // id 가 null 이면 새 엔티티 ([[wrapper-cache-autoboxing]])
    em.persist(entity);
    return entity;
} else {
    return em.merge(entity);              // id 가 있으면 merge
}
```

- **id 가 있는 객체를 save() 하면 merge** → SELECT + 전체 필드 덮어쓰기.
- id 를 직접 할당하는 엔티티는 새 객체도 `isNew() == false` 로 판정돼 merge(SELECT 선행)로 간다 → `Persistable` 로 `isNew()` 를 직접 정의.

## 트레이드오프 / 한계 — 수정에 merge 를 쓰면

```java
// 폼 값으로 새 객체를 만들어 save → id 가 있으니 merge
Post post = new Post(id, form.getTitle(), form.getContent(), form.getCategory());
// author = null, viewCount = 0, likeCount = 0, files = 빈 리스트
postRepository.save(post);
```

| 필드 | merge 후 |
|---|---|
| `viewCount` / `likeCount` | **0** 으로 덮임 |
| `author` | **null** → `not null` 제약이면 플러시 때 위반 → 롤백(수정 기능 마비) |
| `files` | **빈 리스트** → `cascade` 에 MERGE 가 포함되고 `orphanRemoval` 이면 첨부 행 삭제 가능 ([[jpa-cascade-orphan-removal]]) |

- **보안**: merge 는 id 만 맞으면 그 행을 덮어쓴다 → 폼의 숨은 id 를 남의 글 번호로 바꾸면 **남의 글이 수정**된다.

**안전한 수정 방식 — 조회 + 변경 감지**

```java
Post post = findOwned(id, memberId);                                     // 조회 + 소유자 검증
post.update(form.getTitle(), form.getContent(), form.getCategory());     // 바꿀 필드만
// 커밋 시 변경 감지 → UPDATE
```

> ① 엔티티 메서드가 **바꿀 필드만** 바꾼다. ② **조회하는 순간** 소유자 같은 규칙을 검증할 수 있다.
> 프로젝트 크기나 id 생성 전략과는 무관한 선택이다.

> **면접 답변** — "준영속 엔티티는 한때 영속이었고 id도 있지만, 영속성 컨텍스트가 더 이상 관리하지 않는 상태입니다. 트랜잭션이 끝나 영속성 컨텍스트가 닫히거나 detach·clear를 하면 그렇게 되고, 그 뒤의 변경은 반영되지 않습니다. persist는 비영속 엔티티를 영속으로 등록하는 메서드라 준영속에 쓰면 예외가 납니다. merge는 id로 1차 캐시나 DB에서 영속 엔티티를 찾아 모든 필드를 덮어쓴 뒤 그 영속 객체를 반환하고, 넘긴 객체는 준영속 그대로입니다. Spring Data의 save()도 id가 있으면 merge를 호출합니다. 그래서 수정은 merge 대신 조회해서 소유자를 검증하고, 엔티티 메서드로 필요한 필드만 바꿔 변경 감지에 맡깁니다. merge는 폼에 없는 필드까지 덮어써서 조회수나 작성자가 날아갈 수 있기 때문입니다."

> [!WARNING]
> **오답 코너**
> - **"persist 는 대상을 비영속으로 만든다"** — 반대. 비영속을 받아 **영속으로** 만든다. (같은 실수 2회 반복된 항목)
> - **"준영속을 persist 하면 무시된다"** — **예외**. "무시"는 준영속의 setter 결과다.
> - **"준영속에 setter 를 하면 삭제 상태가 된다"** — 상태는 그대로. 삭제는 영속 엔티티에 `remove()` 할 때만.
> - **"merge 는 넘긴 객체를 복사해서 영속으로 바꿔 준다"** — id 로 영속 엔티티를 **얻고** 거기에 값을 덮어쓴 **다른 객체**를 반환.
> - **"merge 방식의 문제는 숫자가 0 이 되는 것"** — author null(제약 위반), files 비움(첨부 삭제 가능), **남의 행 수정**까지.
> - **"merge 를 안 쓰는 이유는 프로젝트가 작아서 / IDENTITY 라서"** — ① 바꿀 필드만, ② 조회 시 소유자 검증.

## 복습 체크

- [ ] 네 가지 상태와 각 상태로 바뀌는 계기를 댈 수 있는가?
- [ ] 영속과 준영속에서 같은 setter 의 결과가 다른 이유는?
- [ ] 준영속을 persist 하면? 준영속에 setter 를 하면?
- [ ] merge 의 동작 순서 세 단계와, `detached == merged` 의 결과는?
- [ ] Spring Data save() 는 언제 persist, 언제 merge 를 부르는가?
- [ ] 폼 값으로 만든 객체를 save() 하면 어떤 필드가 어떻게 되는가?
- [ ] merge 대신 조회 + 변경 감지로 수정하는 이유 **두 가지**를 본인 말로 말할 수 있는가?

## 관련

[[persistence-context]] · [[dirty-checking]] · [[jpa-cascade-orphan-removal]] · [[jpa-association-owner]] · [[jpa-id-generation-strategy]] · [[jpa-bulk-update-and-clear]] · [[open-in-view-osiv]] · [[wrapper-cache-autoboxing]]
