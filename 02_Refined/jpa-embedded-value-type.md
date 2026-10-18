---
type: refined
slug: jpa-embedded-value-type
tags: [값타입, 값 타입, value-type, 임베디드타입, embedded-type, Embedded, Embeddable, 값객체, value-object, 응집도, cohesion, 불변식, invariant, 불변성, immutability, 생성자검증, 앨리어싱, aliasing, 공유참조, setter금지, 새객체로교체, 값타입vs엔티티, value-type-vs-entity, 식별자, identity, ElementCollection, CollectionTable, AttributeOverrides, 모든필드null, JPA, Hibernate, 면접답변]
topic: @Embedded 값 타입은 주인 테이블의 컬럼으로 저장되므로 이점은 객체 쪽(응집도·불변식)에 있다. 공유 시 앨리어싱 때문에 불변으로 두며, 식별자와 개수로 엔티티와 가른다
summary: @Embeddable 필드는 별도 테이블 없이 주인 엔티티 테이블의 컬럼으로 펼쳐져 DB 차이가 없고, 얻는 것은 응집도(데이터와 행동을 한 클래스에)와 불변식 보장(생성자 검증으로 잘못된 조합 차단)이다. 식별자가 없어 같은 인스턴스를 두 엔티티가 공유한 채 setter 로 고치면 양쪽이 바뀌고 UPDATE 가 둘 나가므로(앨리어싱), setter 없이 새 객체로 교체하는 불변 설계로 막는다. 값 타입과 엔티티는 식별자로 하나씩 가리킬 일이 있는지, 주인당 몇 개인지로 가른다. @ElementCollection 은 여러 값을 담지만 식별자가 없어 수정 비용이 크고 개별로 가리킬 수 없다.
contributors: [dongju]
source_refs:
  - https://jakarta.ee/specifications/persistence/3.1/
  - https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#embeddables
updated: 2026-10-18
---

# @Embedded 값 타입

## 개념 정의

```java
@Entity
public class Member {
    @Embedded private NotificationPreference notificationPreference = NotificationPreference.createDefault();
    @Embedded private ProfileImage profileImage;
}

@Embeddable
public class ProfileImage {
    @Column(name = "profile_image_stored_name") private String storedName;
    @Column(name = "profile_image_content_type") private String contentType;

    public ProfileImage(String storedName, String contentType) {
        if (storedName == null || storedName.isBlank()) throw new IllegalArgumentException("저장된 파일 이름이 필요합니다.");
        ...
    }
}
```

- `@Embeddable` 필드는 **주인 엔티티 테이블의 컬럼**으로 펼쳐진다. 별도 테이블도, `@Id` 도 없다. 컬럼 이름 충돌을 피하려고 접두어를 붙이거나 `@AttributeOverrides` 를 쓴다.
- **DB 가 같으니 이점은 전부 객체 쪽**에 있다.

## 작동 원리

### 묶어서 얻는 것

1. **응집도** — 데이터와 그 데이터를 다루는 행동(`notificationPreference.remindsAt(now, startAt)`)을 한 클래스에. 묶지 않으면 로직이 이미 필드가 많은 주인 엔티티로 들어가 비대해진다.
2. **불변식(invariant) 보장** — 생성자에서 검증하니 "파일 이름은 비었는데 형식만 있는" 잘못된 조합이 **존재할 수 없다**. 필드로 흩어 두면 막을 곳이 없다.

### 공유하면 — 앨리어싱

setter 가 있는 값 타입을 두 엔티티가 공유하면:

```java
NotificationPreference pref = memberA.getNotificationPreference();
memberB.setNotificationPreference(pref);   // 같은 인스턴스
pref.setCommentEnabled(false);             // A 만 끄려 했다
```

```
memberA ──┐
          ├──▶ [ 객체 #1 ] commentEnabled = false
memberB ──┘
```

- B 에서 읽어도 false. 변경 감지는 엔티티별로 스냅샷을 비교하므로 **UPDATE 2개** — B 의 설정까지 조용히 바뀐다.
- 엔티티는 id 로 구분돼 공유돼도 "같은 엔티티"지만, **값 타입은 공유되는 순간 남의 값을 바꾸는 부작용**이 된다 ([[shallow-immutability-defensive-copy]]).

### 막는 법 — 불변 + 교체

```java
this.notificationPreference = notificationPreference.change(reminderLeadMinutes, toggles);   // 새 객체 반환
this.profileImage = newImage;                                                                // 통째로 교체
```

```
memberA ──▶ [ 객체 #2 ]   ← 새 객체
memberB ──▶ [ 객체 #1 ]   그대로
```

> setter 없이 새 객체로 교체하면 아무도 공유된 객체 자체를 바꿀 수 없어, 한쪽 변경이 다른 참조자에게 번지지 않는다 ([[string-immutability]] 와 같은 원리).

| | 뜻 | 값 타입에서 |
|---|---|---|
| **불변식** (invariant) | 항상 지켜야 하는 규칙 | 생성자 검증 → **묶는 이유** |
| **불변성** (immutability) | 만든 뒤 값이 안 바뀜 | setter 금지·교체 → **공유 문제의 해법** |

## 트레이드오프 / 한계 — 값 타입 vs 엔티티

같은 필드(`storedName`, `contentType`)를 들고도 프로필 이미지는 값 타입, 첨부파일은 엔티티일 수 있다.

| 질문 | 값 타입 (`ProfileImage`) | 엔티티 (`AttachedFile`) |
|---|---|---|
| 식별자가 필요한가? | 없음. 값이 같으면 같음, 통째로 교체 | id 로 **하나만 골라** 삭제·다운로드 |
| 몇 개인가? | 주인당 하나 → 컬럼 | 여러 개 → 별도 테이블 + FK |
| 생명주기 | 주인과 완전히 함께 | 자기 생명주기 (cascade 로 묶음, [[jpa-cascade-orphan-removal]]) |
| 공유 | 앨리어싱 위험 → 불변 | id 로 구분 |

- `@ElementCollection` 은 값 타입 여러 개를 별도 테이블(`@CollectionTable`)에 담지만, 식별자가 없어 하나만 바꿔도 전체 삭제 후 재삽입이 일어나기 쉽고 개별로 가리킬 수 없다 → 개별로 다뤄야 하면 엔티티.
- 모든 필드가 null 인 `@Embedded` 는 Hibernate 가 null 로 읽는다. 빈 객체가 들어오는 경우까지 대비하려면 `isPresent()` 같은 확인 메서드를 둔다.
- 값 타입의 `equals`/`hashCode` 는 id 가 아니라 **모든 필드**로 비교해야 한다 ([[jpa-entity-equality]] 와 대비).

> **면접 답변** — "@Embedded 값 타입은 별도 테이블 없이 주인 엔티티의 테이블에 컬럼으로 저장됩니다. 그래서 얻는 건 객체 쪽에 있습니다. 데이터와 그 데이터를 다루는 로직을 한 클래스에 모아 응집도를 높이고, 생성자 검증으로 잘못된 상태가 아예 만들어지지 않게 합니다. 값 타입은 setter 없이 불변으로 두고 바꿀 때 새 객체로 교체하는데, 같은 인스턴스를 두 엔티티가 공유한 채 값을 고치면 다른 엔티티까지 바뀌는 앨리어싱 문제가 생기기 때문입니다. 엔티티와의 차이는 식별자와 개수입니다. 회원당 하나이고 통째로 교체하는 프로필 이미지는 값 타입으로, 글당 여러 개이고 id로 하나씩 지우고 내려받는 첨부파일은 엔티티로 만들었습니다."

> [!WARNING]
> **오답 코너**
> - **"값 타입으로 묶으면 엔티티 구조와 로직이 간략해진다"** — 이름을 붙일 것: **응집도**, **불변식 보장**.
> - **"공유된 값 타입을 setter 로 고치면 A 만 바뀐다"** — 같은 객체라 B 도 바뀌고 UPDATE 2개.
> - **"그 현상은 영속성 전이"** — **앨리어싱**. JPA 와 무관한 자바 참조 문제. 영속성 전이는 cascade.
> - **"프로필 이미지는 경로만 업데이트"** — setter 없음. 새 객체로 **통째로 교체**.
> - **"묶는 이유는 불변성"** — 묶는 이유는 **불변식**. 불변성은 공유 문제의 해법. 비슷하게 들리지만 다른 개념.

## 복습 체크

- [ ] `@Embeddable` 필드는 DB 어디에 저장되는가?
- [ ] DB 가 같은데 값 타입으로 묶는 이유 두 가지는?
- [ ] 값 타입을 공유한 채 setter 로 고치면 무슨 일이 생기는지 메모리 그림과 UPDATE 수로 설명할 수 있는가?
- [ ] 그 현상의 이름과, 막는 설계는?
- [ ] 불변식과 불변성을 구분해 말할 수 있는가?
- [ ] 같은 필드를 가진 두 클래스를 값 타입 / 엔티티로 가르는 기준 두 가지는?
- [ ] `@ElementCollection` 의 한계는?

## 관련

[[jpa-entity-default-constructor]] · [[jpa-cascade-orphan-removal]] · [[jpa-entity-equality]] · [[shallow-immutability-defensive-copy]] · [[string-immutability]] · [[dirty-checking]]
