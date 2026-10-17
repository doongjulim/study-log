---
date: 2026-10-17
type: raw
author: dongju
tags: [엔티티 생명주기, entity lifecycle, 엔티티 상태, 비영속, transient, new, 영속, managed, 준영속, detached, 삭제, removed, persist, merge, detach, clear, remove, PersistentObjectException, detached entity passed to persist, 1차 캐시, 변경 감지, dirty checking, 지연 로딩, OSIV, SimpleJpaRepository, save, isNew, Spring Data JPA]
topic: 엔티티의 네 가지 상태와 전이 — persist 는 비영속을 영속으로 등록(준영속이면 예외), merge 는 id 로 영속 엔티티를 얻어 값을 덮어쓴 다른 객체를 반환, Spring Data save() 는 id 가 있으면 merge
summary: 엔티티는 비영속 → persist → 영속 → 트랜잭션 종료·detach·clear → 준영속, 영속에서 remove → 삭제 상태가 된다. 준영속은 id 는 있지만 영속성 컨텍스트가 관리하지 않아 변경 감지·지연 로딩이 없고 setTitle 은 무시된다. persist 는 비영속 객체 자체를 영속으로 등록하며 준영속에 쓰면 PersistentObjectException 이 난다. merge 는 인자의 id 로 1차 캐시나 DB(SELECT)에서 영속 엔티티를 얻어 모든 필드를 덮어쓴 뒤 그 영속 객체를 반환하고, 인자는 준영속 그대로다. Spring Data save() 는 isNew(id null)면 persist, 아니면 merge 를 호출한다.
source: session
distilled: true
distilled_at: 2026-10-17
---

## 배운 개념

### 1. 네 가지 상태

```java
Post post = new Post("제목", ...);      // ① 비영속 — 영속성 컨텍스트와 무관한 자바 객체

@Transactional
void save() {
    em.persist(post);                   // ② 영속 — persist 로 등록
    post.setTitle("새 제목");            //    → 커밋 때 스냅샷과 비교해 UPDATE
}                                       // 트랜잭션 종료 (OSIV off → 영속성 컨텍스트도 종료)

post;                                   // ③ 준영속
post.setTitle("또 바꿈");                // ④ 여전히 준영속 — 무시됨, DB 반영 X
```

| 상태 | 들어가는 계기 |
|---|---|
| 비영속 | `new` |
| 영속 | `persist`, `find`/조회, `merge` 의 반환값 |
| 준영속 | 트랜잭션 종료(영속성 컨텍스트 종료), `em.detach(entity)`, `em.clear()` |
| 삭제 | 영속 엔티티에 `em.remove()` → 커밋 때 DELETE |

> **②와 ④의 차이 = 영속성 컨텍스트가 변경 감지를 해 주느냐.** 영속 상태는 스냅샷과 비교해 커밋 때 UPDATE 를 만들고, 준영속은 비교하지 않아 UPDATE 를 만들지 않는다.

- **준영속** = 한때 영속이었고 **id 도 있지만** 지금은 관리받지 못하는 상태. 변경 감지·지연 로딩 없음.

### 2. persist — 준영속에 쓰면 예외

```java
@Transactional
void retry(Post detached) {   // id = 1
    em.persist(detached);     // → PersistentObjectException: detached entity passed to persist
}                             //   (스프링에선 InvalidDataAccessApiUsageException 으로 감싸짐)
```

- persist 는 **비영속** 엔티티를 **새로 등록**(INSERT)하는 메서드. 이미 id=1 행이 있는데 또 INSERT 할 수는 없다 → Hibernate 가 id 가 채워진 엔티티를 준영속으로 판단하고 미리 막는다.

### 3. merge — 다른 객체를 돌려준다

```java
@Transactional
void update(Post detached) {               // detached: id=1, title="또 바꿈"
    Post merged = em.merge(detached);
    System.out.println(detached == merged); // false
    detached.setTitle("A");                 // 무시 (여전히 준영속)
    merged.setTitle("B");                   // 변경 감지 → DB title = "B"
}
```

```
em.merge(detached)
  → id=1 로 1차 캐시 조회 → 없음
  → select * from post where id = 1        ← DB 에서 조회해 영속 엔티티(merged)를 만든다
  → detached 의 필드 값을 merged 에 "모두" 덮어쓴다
  → merged 를 반환 (영속)
detached → 여전히 준영속
```

- 흔한 실수: `em.merge(detached);` 처럼 **반환값을 버리고** 계속 `detached` 를 수정 → 이후 수정은 전부 사라진다.

| 메서드 | 받는 상태 | 하는 일 | 반환 |
|---|---|---|---|
| `persist` | 비영속 | 그 객체 **자체**를 영속으로 등록 → INSERT. 준영속이면 **예외** | void |
| `merge` | 준영속(또는 비영속) | id 로 1차 캐시 → 없으면 SELECT → 영속 엔티티에 **모든 필드 덮어씀** | **다른** 영속 객체 |

### 4. Spring Data `save()` 의 정체

```java
// SimpleJpaRepository.save()
if (entityInformation.isNew(entity)) {   // id 가 null 이면 새 엔티티
    em.persist(entity);
    return entity;
} else {
    return em.merge(entity);              // id 가 있으면 merge
}
```

- **id 가 있는 객체를 `save()` 하면 merge** → SELECT + 전체 필드 덮어쓰기.

## 오답·헷갈린 점

- **전**: 준영속에 `setTitle` 한 ④ = "삭제" 상태 → **후**: 필드 변경은 상태를 안 바꾼다. 여전히 **준영속**. 삭제는 영속 엔티티에 `remove()` 할 때만.
- **전**: persist 는 "비영속 상태로 변경한다" (15:15) → 최종 답변에서도 "대상을 비영속 상태로 만든다" (18:42) → **후**: 비영속을 받아 **영속으로** 만든다. ⚠️ **같은 실수 2회 반복** — 방향을 거꾸로 기억하고 있음.
- **전**: 준영속을 persist 하면 "무시된다" (18:45) → **후**: **예외**. 15:18 에는 스스로 "예외"라고 맞혔는데 나중에 "무시"(준영속 setTitle 의 결과)와 섞임.

| 상황 | 결과 |
|---|---|
| 준영속에 `setTitle(...)` | **무시** (변경 감지 없음) |
| 준영속을 `persist(...)` | **예외** |

- **전**: merge 는 "객체를 복사해서 영속 상태로 바꿔서 돌려준다" → **후**: 인자를 복사하는 게 아니라, id 로 **영속 엔티티를 얻고** 거기에 **값을 덮어쓴** 그 객체를 반환.
- **전**: 준영속 = "insert, update 되지 않는 상태" → **후**: "id 는 있지만 **변경 감지·지연 로딩을 받지 못하는** 상태".
- ✅ `detached == merged` false, DB 결과 B, 준영속 전환 원인(트랜잭션 종료)은 정확히 답함.

## Q&A

- **Q. new / persist / 트랜잭션 종료 후 / 그 뒤 setTitle 의 상태?** A. 비영속 / 영속 / 준영속 / 준영속(변경 무시).
- **Q. 영속과 준영속에서 같은 setTitle 의 결과가 다른 이유?** A. 영속은 스냅샷 비교로 변경 감지, 준영속은 비교 안 함.
- **Q. 준영속을 persist 하면?** A. 예외. persist 는 비영속을 새로 등록하는 메서드.
- **Q. merge 의 반환값과 인자는 같은 객체?** A. 아니다(false). 반환값만 영속.
- **Q. Spring Data save() 는 언제 merge?** A. id 가 있을 때(isNew false).

## 다음 학습

- `Persistable` 로 `isNew()` 직접 정의 — id 를 직접 할당하는 엔티티를 save() 하면 merge 로 가서 SELECT 가 먼저 나가는 문제
- 연결: [[persistence-context]], [[dirty-checking]], [[jpa-bulk-update-and-clear]], [[open-in-view-osiv]], [[2026-10-17-merge-vs-dirty-checking-update]]
