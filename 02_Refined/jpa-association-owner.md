---
type: refined
slug: jpa-association-owner
tags: [연관관계의주인, 연관관계 주인, owning-side, owner, 비주인, inverse-side, mappedBy, 양방향연관관계, bidirectional, JoinColumn, 외래키, FK, 패러다임불일치, 객체참조, OneToMany, ManyToOne, N쪽, 읽기전용, 연관관계편의메서드, 편의 메서드, convenience-method, addFile, setPost, 주인없는행, FK-null, cascade통로, 비영속, JPA, Hibernate, 면접답변]
topic: 연관관계의 주인 — 양방향에서 객체 참조 2개 vs FK 1개 불일치를 해소하려고 둘 중 FK 를 관리할 필드 하나를 정하는 규칙. mappedBy 의 뜻과, 그럼에도 양쪽을 다 세팅하는 연관관계 편의 메서드
summary: 양방향이면 객체는 서로를 가리키는 참조가 2개인데 테이블 FK 는 1개라, 두 참조 값이 다를 때 FK 에 무엇을 쓸지 하나를 기준으로 정해야 한다. 그 기준 필드가 연관관계의 주인이며 주인만 FK 를 등록·수정하고 비주인은 조회만 한다(우선순위가 아니라 완전히 무시). 보통 FK 가 있는 N 쪽(@JoinColumn)이 주인이다. mappedBy = "post" 는 "이 컬렉션은 상대 클래스의 post 필드에 의해 매핑되므로 주인이 아니다"라는 선언이다. 그래도 비주인 컬렉션은 cascade 의 통로이자 메모리 상태라서, 편의 메서드로 양쪽을 함께 세팅한다.
contributors: [dongju]
source_refs:
  - https://jakarta.ee/specifications/persistence/3.1/
  - https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#associations-one-to-many-bidirectional
updated: 2026-10-17
---

# 연관관계의 주인과 mappedBy

## 개념 정의

### 왜 필요한가 — 참조 2개 vs FK 1개

```java
class Post {
    @OneToMany(mappedBy = "post")
    List<AttachedFile> files;          // 참조 ①  Post → AttachedFile
}
class AttachedFile {
    @ManyToOne @JoinColumn(name = "post_id")
    Post post;                          // 참조 ②  AttachedFile → Post
}
```

```
attached_file ( id, stored_name, post_id )   -- FK 는 하나. 이것 하나로 양방향 조회가 다 된다
```

두 참조는 서로 다른 값을 가질 수 있다.

```java
postA.getFiles().add(file);   // 리스트에는 글 A
file.setPost(postB);          // 필드에는 글 B
// → post_id = B
```

DB 에 쓸 칸은 하나뿐이므로 JPA 는 **둘 중 하나만 믿기로** 정해 둔다.

> **연관관계의 주인** — 양방향 연관관계에서 두 참조 **필드** 중 FK 를 관리하는 쪽. 주인만 FK 를 등록·수정하고, 주인이 아닌 쪽은 연관관계를 조회만 한다.

- 주인은 테이블도, 엔티티 전체도 아니고 **필드** 단위로 정해진다.
- 보통 **FK 가 있는 테이블에 매핑되는 N 쪽(`@JoinColumn`)** 이 주인 → 흔히 생각하는 "부모"가 아니라 **자식** 쪽.
- 비주인의 변경은 FK 를 쓸 때 "우선순위가 낮은" 게 아니라 **완전히 무시**된다. 컬렉션에서 자식을 빼도 SQL 이 0건인 이유 ([[jpa-cascade-orphan-removal]]).
- 비주인 엔티티도 자기 필드(`Post.title`)는 얼마든지 수정된다. 막히는 건 **연관관계(FK)** 뿐.

### mappedBy

```java
class Post {
    @OneToMany(mappedBy = "post")   // "post" 라는 이름의 필드를 원소 타입 AttachedFile 안에서 찾아라
    List<AttachedFile> files;
}
class AttachedFile {
    @ManyToOne @JoinColumn(name = "post_id")
    Post post;                       // ← 여기. 이 필드가 주인
}
```

> **`mappedBy = "post"`** = "이 컬렉션(`Post.files`)은 **`AttachedFile` 클래스의 `post` 필드**에 의해 매핑된다. 따라서 이쪽은 주인이 아니다."

- 값은 **상대 클래스의 필드 이름**이다. 테이블 이름도 컬럼 이름도 아니다.
- 어떤 엔티티를 참조하는지는 타입(`List<AttachedFile>`)이 말해 준다. mappedBy 가 추가로 말하는 건 "나는 거울이고 진짜 매핑은 저쪽에 있다".
- 없는 이름을 적으면 애플리케이션 **구동 시점에** 실패한다.

## 작동 원리 — 연관관계 편의 메서드

```java
public void addFile(AttachedFile file) {
    files.add(file);       // ① 비주인 — cascade 통로 + 메모리 정합성
    file.setPost(this);    // ② 주인 — FK(post_id) 값
}
```

자식을 부모의 cascade PERSIST 로 저장하는 구조(`post.addFile(f); save(post);`)에서 한쪽만 세팅하면:

| 세팅한 쪽 | 결과 | 이유 |
|---|---|---|
| 비주인만 (`files.add`) | INSERT 는 되지만 **`post_id = NULL`** — 다른 컬럼은 다 채워진 "주인 없는 행". 다음 요청에서 안 보임 | FK 는 주인만 보고 쓴다 |
| 주인만 (`setPost`) | **INSERT 없음** — 자식이 비영속 그대로 | cascade 는 `files` **필드를 따라** 전이되는데 컬렉션이 비어 있음 |
| 양쪽 다 (`addFile`) | ✅ | |

- "비주인은 무시된다"는 **FK 를 쓸 때만** 맞다. 비주인 컬렉션은 **cascade 의 통로**이고, 같은 트랜잭션 안의 **메모리 상태**(`post.getFiles().size()` 로 개수 검증 등)를 책임진다.
- 자식을 `fileRepository.save(file)` 로 직접 저장한다면 주인만 세팅해도 DB 는 맞다 — 대신 그 트랜잭션 안에서 `post.getFiles()` 가 실제와 달라진다.

## 트레이드오프 / 한계

- 편의 메서드를 **양쪽 엔티티 모두**에 두면 서로를 호출하며 무한 루프가 날 수 있다 → 한쪽(보통 부모)에만 둔다.
- `post_id` 에 `not null` 제약이 없으면 "주인 없는 행" 버그를 DB 가 막아 주지 못한다 → 자식의 `@JoinColumn(nullable = false)` + DB 제약을 검토.
- `@OneToMany` **단방향** + `@JoinColumn`(주인이 1 쪽)은 INSERT 후 FK 를 채우는 추가 UPDATE 가 나간다 → FK 가 있는 N 쪽을 주인으로 두는 이유.

> **면접 답변** — "양방향 연관관계에서 객체는 서로를 가리키는 참조가 두 개인데, 테이블은 FK가 하나뿐입니다. 두 참조 값이 다를 때 FK에 무엇을 쓸지 정하려면 둘 중 하나를 기준으로 삼아야 하고, 그게 연관관계의 주인입니다. 주인만 FK를 등록·수정하고, 주인이 아닌 쪽은 조회만 합니다. 보통 FK가 있는 N 쪽이 주인이 됩니다. mappedBy = "post"는 '나는 주인이 아니고, AttachedFile의 post 필드에 의해 매핑된다'는 선언입니다. 그렇다고 한쪽만 세팅하면 안 돼서 편의 메서드에서 양쪽을 함께 세팅합니다. 비주인 리스트만 넣으면 cascade로 저장은 되지만 FK가 null인 행이 생기고, 주인만 세팅하면 cascade가 컬렉션을 따라 전이되기 때문에 자식이 아예 저장되지 않습니다."

> [!WARNING]
> **오답 코너**
> - **"객체 참조 1개, 테이블 PK 3개"** — 객체 참조 **2개**, 테이블 **FK 1개**.
> - **"리스트(A)와 필드(B)가 다르면 A 가 들어간다"** — **B**. 주인(`@JoinColumn`)만 본다.
> - **"주인을 정하려면 부모 자식 관계가 필요하다"** — 필요한 건 "어느 참조가 FK 를 관리하나". 주인은 오히려 자식(N) 쪽.
> - **"두 테이블의 주인", "주인은 두 테이블 모두 수정"** — 주인은 **객체의 참조 필드**이고, 수정하는 건 **FK 값** 하나다. 테이블에는 FK 가 하나라 주인을 정할 일이 없다.
> - **"mappedBy="post" = post 의 files 필드에 의해 매핑"** — mappedBy 가 붙은 쪽이 `files` 자신이다. "**AttachedFile** 의 **post** 필드에 의해".
> - **"mappedBy 는 어떤 엔티티를 참조하는지 명시한다"** — 그건 타입이 한다.
> - **"주인만 세팅 → 가비지 행 / 비주인만 세팅 → 비영속"** — **반대**. 주인만 → 비영속(INSERT 없음), 비주인만 → FK NULL 행.

## 복습 체크

- [ ] 양방향에서 객체 참조와 FK 의 수를 대고, 그 불일치가 왜 "주인"을 필요하게 만드는지 설명할 수 있는가?
- [ ] 리스트엔 A, 필드엔 B 를 넣으면 FK 에 무엇이 들어가는가?
- [ ] 연관관계의 주인을 "필드", "FK", "조회만" 을 넣어 정의할 수 있는가?
- [ ] 보통 어느 쪽이 주인이 되며, 그게 "부모"가 아닌 이유는?
- [ ] `mappedBy = "post"` 의 "post" 는 어느 클래스의 무엇인가?
- [ ] 편의 메서드에서 주인만 / 비주인만 세팅하면 각각 어떻게 되는가?
- [ ] "비주인은 무시된다"가 맞는 경우와 틀리는 경우를 구분할 수 있는가?

## 관련

[[jpa-cascade-orphan-removal]] · [[jpa-entity-lifecycle-merge]] · [[jpa-entity-equality]] · [[persistence-context]] · [[dirty-checking]]
