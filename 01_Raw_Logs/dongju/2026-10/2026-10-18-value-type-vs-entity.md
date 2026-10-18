---
date: 2026-10-18
type: raw
author: dongju
tags: [값 타입, value type, 엔티티, entity, 값 타입 vs 엔티티, Embedded, Embeddable, 식별자, identity, id, 생명주기, lifecycle, 1:N, 별도 테이블, ElementCollection, CollectionTable, 컬렉션 값 타입, 전체 삭제 후 재삽입, AttributeOverrides, ProfileImage, AttachedFile, deleteFile, 모든 필드 null, study-board, 면접 답변]
topic: 값 타입과 엔티티를 가르는 기준 — 식별자로 하나씩 가리킬 일이 있나, 주인당 몇 개인가. ProfileImage(값 타입)와 AttachedFile(엔티티)은 같은 필드를 들고도 다르게 설계됐다
summary: ProfileImage 와 AttachedFile 은 둘 다 저장된 파일 이름과 형식을 들지만, 프로필 이미지는 회원당 하나이고 통째로 교체하면 되므로 member 테이블 컬럼(값 타입)으로 두었고, 첨부파일은 글당 여러 개인 데다 deleteFile(postId, fileId) 처럼 id 로 하나씩 골라 삭제·다운로드해야 하므로 별도 테이블의 엔티티로 만들었다. 값 타입은 식별자가 없고 생명주기가 주인과 완전히 같으며, 공유 시 앨리어싱 때문에 불변으로 둔다. @ElementCollection 은 값 타입 여러 개를 별도 테이블에 담지만 식별자가 없어 수정 시 전체 삭제 후 재삽입이 일어나기 쉽고 개별로 가리킬 수 없다.
source: session
distilled: true
distilled_at: 2026-10-18
---

## 배운 개념

### 1. 같은 필드, 다른 설계

```java
// 값 타입 — member 테이블 컬럼
@Embeddable
public class ProfileImage {
    private String storedName;
    private String contentType;
}

// 엔티티 — attached_file 테이블
@Entity
public class AttachedFile {
    @Id @GeneratedValue Long id;
    private String storedName;
    private String contentType;
    @ManyToOne Post post;
}
```

### 2. 가르는 기준

| 질문 | 값 타입 (`ProfileImage`) | 엔티티 (`AttachedFile`) |
|---|---|---|
| **식별자가 필요한가?** | 없음. 값이 같으면 같은 것, 통째로 교체 | `id` 로 **하나만 골라** 지우고 내려받아야 함 (`deleteFile(postId, fileId)`) |
| **몇 개인가?** | 주인당 하나 → 주인 테이블의 컬럼으로 펼침 | 주인당 여러 개 → 별도 테이블 + FK |
| **생명주기** | 주인과 완전히 함께. `null` 로 바꾸면 컬럼 NULL | 자기 생명주기 (cascade·orphanRemoval 로 묶음) |
| **공유** | 공유하면 앨리어싱 위험 → 불변으로 막음 | id 로 구분되니 공유돼도 "같은 엔티티" |

- 컬럼으로 펼치는 방식으로는 "여러 개"를 담을 수 없다 (컬럼 수가 고정).

### 3. 중간 선택지 — @ElementCollection

- 값 타입 **여러 개**를 별도 테이블(`@CollectionTable`)에 저장.
- 식별자가 없어 하나만 바꿔도 **전부 지우고 다시 INSERT** 하는 경우가 많고, 하나를 골라 가리킬 방법이 없다.
- 첨부파일처럼 개별로 다뤄야 하면 결국 엔티티가 맞다.

### 4. 기타

- 모든 필드가 null 인 `@Embedded` 는 Hibernate 가 **null** 로 읽는다. 구현에 따라 빈 객체가 들어오는 경우를 대비해 study-board 는 `ProfileImage.isPresent()` 와 `Member.hasProfileImage()` 로 한 번 거른다.
- 같은 값 타입을 한 엔티티에 두 번 쓰면 컬럼 이름이 겹친다 → `@AttributeOverrides`.

### 5. 면접 답변 (30초)

> "@Embedded 값 타입은 별도 테이블 없이 주인 엔티티의 테이블에 컬럼으로 저장됩니다. 그래서 DB 차이는 없고, 얻는 건 객체 쪽에 있습니다. 첫째, 알림 설정 데이터와 '언제 알릴지' 판단하는 로직을 한 클래스에 모아 응집도를 높였고, Member가 비대해지지 않았습니다. 둘째, 생성자에서 검증해서 '파일 이름 없는 프로필 이미지' 같은 잘못된 상태가 아예 만들어지지 않게 했습니다.
> 값 타입은 setter 없이 불변으로 만들고, 바꿀 때는 새 객체로 교체합니다. 같은 인스턴스를 두 엔티티가 공유한 상태에서 값을 고치면 다른 엔티티까지 바뀌는 앨리어싱 문제가 생기기 때문입니다.
> 엔티티와의 차이는 식별자와 개수입니다. 프로필 이미지는 회원당 하나이고 통째로 교체하면 되니 값 타입으로 두었고, 첨부파일은 글당 여러 개인 데다 id로 하나씩 골라 삭제하고 내려받아야 해서 별도 엔티티로 만들었습니다."

## 오답·헷갈린 점

- ✅ 개수(1개 vs 여러 개 → 컬럼으로는 여러 개 불가)와 "프로필 이미지는 id 로 가리킬 일이 없다"는 스스로 정확히 답함.
- **전**: 프로필 이미지는 "이미지 경로만 계속 업데이트" → **후**: 새 객체로 **통째로 교체**.
- **전**: 최종 답변에서 첨부파일이 엔티티인 이유 = "여러 개라서"만 → **후**: + **id 로 하나씩 골라 삭제·다운로드**해야 해서. 식별자 필요 여부가 값 타입/엔티티를 가르는 첫 번째 기준.

## Q&A

- **Q. ProfileImage 와 AttachedFile 은 같은 필드인데 왜 하나는 값 타입, 하나는 엔티티?** A. 개수(1 vs N)와 식별자 필요 여부(통째 교체 vs id 로 개별 삭제·다운로드).
- **Q. 값 타입 여러 개가 필요하면?** A. @ElementCollection 이 있지만 식별자가 없어 수정 비용이 크고 개별 지칭 불가 → 개별로 다루면 엔티티.

## 다음 학습

- @ElementCollection 의 전체 삭제 후 재삽입을 `show-sql` 로 직접 확인
- 연결: [[2026-10-18-embedded-value-type-why]], [[jpa-cascade-orphan-removal]], [[jpa-entity-lifecycle-merge]]
