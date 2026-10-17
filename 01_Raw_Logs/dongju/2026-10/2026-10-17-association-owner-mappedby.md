---
date: 2026-10-17
type: raw
author: dongju
tags: [연관관계의 주인, owning side, 주인, owner, 비주인, inverse side, mappedBy, 양방향 연관관계, bidirectional, JoinColumn, 외래 키, FK, 패러다임 불일치, 객체 참조, OneToMany, ManyToOne, N 쪽, 읽기 전용, read-only, 필드 이름, Post, AttachedFile, study-board, 면접 답변]
topic: 연관관계의 주인은 양방향에서 두 참조 필드 중 FK 를 관리하는 쪽 — 객체 참조 2개 vs FK 1개 불일치를 해소하는 규칙, mappedBy 는 "상대 클래스의 그 필드가 주인"이라는 선언
summary: 양방향이면 객체는 서로를 가리키는 참조가 2개(Post.files, AttachedFile.post)인데 테이블 FK 는 1개(attached_file.post_id)라, 두 참조 값이 다를 때 FK 에 무엇을 쓸지 하나를 기준으로 정해야 한다. 그 기준이 연관관계의 주인이며, 주인만 FK 를 등록·수정하고 비주인은 조회만 한다(우선순위가 아니라 완전히 무시). 보통 FK 가 있는 N 쪽(@JoinColumn)이 주인이다. mappedBy = "post" 는 "이 컬렉션은 AttachedFile 의 post 필드에 의해 매핑되므로 주인이 아니다"라는 선언으로, 값은 상대 클래스의 필드 이름이다.
source: session
distilled: false
---

## 배운 개념

### 1. 왜 주인이 필요한가 — 참조 2개 vs FK 1개

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
attached_file ( id, stored_name, post_id )   -- FK 는 post_id 하나
```

- **객체: 참조 2개**, **테이블: FK 1개**.
- 테이블은 FK 하나로 양쪽 조회가 다 된다.

```sql
select * from attached_file where post_id = ?      -- 글 → 파일
select * from post where id = (파일의 post_id)       -- 파일 → 글
```

- 두 참조가 서로 다른 값을 가질 수 있다.

```java
postA.getFiles().add(file);   // 리스트에는 글 A
file.setPost(postB);          // 필드에는 글 B
// → post_id = B. JPA 는 주인(@JoinColumn 쪽)만 보고 FK 를 쓴다
```

- 증거: 10/16 세션에서 orphanRemoval 없이 `post.getFiles().remove(target)` 만 하면 **SQL 0건**이었다. 비주인 리스트 변경은 FK 에 반영되지 않는다.
- 비주인 쪽은 "우선순위가 낮은" 게 아니라 FK 를 쓸 때 **완전히 무시**된다.

### 2. 연관관계의 주인 — 정의

> **양방향 연관관계에서 두 참조 필드 중 FK 를 관리하는 쪽. 주인만 FK 를 등록·수정할 수 있고, 주인이 아닌 쪽은 연관관계를 조회만 한다.**

- 주인은 테이블도, 엔티티 전체도 아니고 **필드** 단위. study-board 에서는 `AttachedFile.post` 필드.
- 보통 **FK 가 있는 테이블에 매핑되는 쪽(N 쪽, `@JoinColumn`)** 이 주인 → 흔히 생각하는 "부모"(Post)가 아니라 **자식** 쪽.
- 주인이 수정하는 건 **FK 값** 하나. 비주인 엔티티도 자기 필드(`Post.title` 등)는 얼마든지 수정 가능 — 막히는 건 연관관계(FK)뿐.
- 테이블에는 FK 가 하나라 주인을 정할 일이 없다. 주인이 필요한 건 참조가 둘인 **객체 쪽**.

### 3. mappedBy — 뜻

```java
class Post {
    @OneToMany(mappedBy = "post")   // ① "post" 라는 이름의 필드를 찾아라
    List<AttachedFile> files;        //    어디서? ← 원소 타입인 AttachedFile 안에서
}

class AttachedFile {
    @ManyToOne
    @JoinColumn(name = "post_id")
    Post post;                       // ② 여기 있다. 이 필드가 주인
}
```

> **`mappedBy = "post"` = "이 컬렉션(`Post.files`)은 `AttachedFile` 클래스의 `post` 필드에 의해 매핑된다. 따라서 이쪽은 주인이 아니다."**

- 값은 **상대 클래스의 필드 이름**. 테이블 이름도 컬럼 이름도 아니다.
- 어떤 엔티티를 참조하는지는 타입(`List<AttachedFile>`)이 말해 준다. `mappedBy` 가 추가로 말하는 건 "나는 거울이고, 진짜 매핑(`@JoinColumn`)은 저쪽에 있다".
- 존재하지 않는 이름(`mappedBy = "writer"`)을 적으면 애플리케이션 **구동 시점에** 실패.

## 오답·헷갈린 점

- **전**: 객체 참조 1개, 테이블 **PK** 3개 → **후**: 객체 참조 **2개**, 테이블 **FK 1개**. 질문은 FK 였는데 PK·컬럼 수를 셈.
- **전**: 리스트(A)와 필드(B)가 다르면 **A** 가 들어간다 → **후**: **B**. 주인(`@JoinColumn`)만 본다. 전날 "리스트에서 빼도 SQL 0건"과 모순이었음.
- **전**: 주인을 정하려면 "부모 자식 관계가 정의돼야 한다" → **후**: 필요한 건 "어느 참조가 FK 를 관리하나". 주인은 오히려 자식(N) 쪽.
- **전**: JoinColumn 쪽을 "우선적으로" 본다 → **후**: 우선순위가 아니라 비주인은 **완전히 무시**.
- **전**: "두 **테이블**의 주인", "주인 **테이블**" → **후**: 주인은 **객체의 참조 필드**.
- **전**: 주인은 "두 테이블 모두 수정", 비주인은 "주인을 읽기만" → **후**: 주인이 수정하는 건 **FK 값** 하나, 비주인은 **연관관계**를 조회만.
- **전**: 정의에서 "두 **엔티티** 중" → **후**: "두 참조 **필드** 중". 같은 엔티티의 다른 연관 필드는 주인 여부가 따로 정해진다.
- **전**: `mappedBy = "post"` = "**post 의 files** 필드에 의해 매핑" (힌트 후에도 2회 반복) → **후**: "**AttachedFile** 의 **post** 필드에 의해". mappedBy 가 붙은 쪽이 `files` 자신이므로 자기 자신에 의해 매핑될 수 없다.
- **전**: mappedBy 는 "어떤 엔티티의 객체를 참조하는지 명시" → **후**: 그건 타입이 함. mappedBy 는 "나는 주인이 아니고 저 필드가 주인"이라는 선언.
- ⚠️ 반복 테마: **객체 레벨(필드·참조) vs DB 레벨(테이블·FK) 혼동** — 10/13 프록시를 "조인"으로, 10/16 cascade 를 "테이블 CRUD"로, 이번 주인을 "테이블"로. 3세션 연속. 답변할 때 "테이블"이라는 단어가 나오면 객체 얘기인지 DB 얘기인지 스스로 점검할 것.

## Q&A

- **Q. 양방향에서 객체 참조와 FK 의 수는?** A. 참조 2개, FK 1개.
- **Q. 리스트엔 A, 필드엔 B 면 post_id 는?** A. B. 주인(`@JoinColumn`)만 본다.
- **Q. 연관관계의 주인 정의?** A. 두 참조 필드 중 FK 를 관리하는 쪽. 주인만 FK 등록·수정, 비주인은 조회만.
- **Q. 누가 주인이 되나?** A. FK 가 있는 N 쪽. study-board 는 `AttachedFile.post`.
- **Q. `mappedBy = "post"` 의 "post" 는?** A. 상대 클래스(AttachedFile)의 필드 이름.

## 다음 학습

- `@OneToMany` **단방향** + `@JoinColumn` (주인이 1 쪽) → INSERT 후 FK 를 채우는 추가 UPDATE 가 나가는 이유
- 연결: [[2026-10-17-bidirectional-convenience-method]], [[2026-10-16-cascade-vs-orphanremoval]]
