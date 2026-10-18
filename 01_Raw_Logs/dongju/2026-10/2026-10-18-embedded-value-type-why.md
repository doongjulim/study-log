---
date: 2026-10-18
type: raw
author: dongju
tags: [값 타입, value type, 임베디드 타입, embedded type, Embedded, Embeddable, 값 객체, value object, 응집도, cohesion, 불변식, invariant, 불변성, immutability, 생성자 검증, 앨리어싱, aliasing, 공유 참조, setter 금지, 새 객체로 교체, 변경 감지, dirty checking, NotificationPreference, ProfileImage, Member, study-board]
topic: @Embedded 값 타입은 주인 테이블의 컬럼으로 저장되므로 이점은 객체 쪽에 있다 — 응집도와 불변식 보장. 공유 시 앨리어싱이 생기므로 setter 없이 새 객체로 교체(불변)한다
summary: @Embeddable 클래스의 필드는 별도 테이블 없이 주인 엔티티 테이블(member)의 컬럼으로 저장되어 DB 차이는 없다. 묶어서 얻는 것은 ① 응집도(알림 설정 데이터와 remindsAt 같은 행동을 한 클래스에, Member 비대화 방지) ② 불변식 보장(ProfileImage 생성자가 빈 파일 이름을 거부해 잘못된 조합이 존재할 수 없음)이다. 값 타입은 id 가 없어 같은 인스턴스를 두 엔티티가 공유하면 setter 수정이 양쪽에 번지고(앨리어싱) 변경 감지로 UPDATE 가 둘 나가므로, setter 없이 change() 가 새 객체를 반환해 교체하는 불변 설계로 막는다.
source: session
distilled: false
---

## 배운 개념

### 1. study-board 의 값 타입

```java
// Member.java
@Embedded
private NotificationPreference notificationPreference = NotificationPreference.createDefault();

@Embedded
private ProfileImage profileImage;   // 없으면 화면이 닉네임 첫 글자로 대신 그린다
```

```java
@Embeddable
public class ProfileImage {
    @Column(name = "profile_image_stored_name", length = 100)
    private String storedName;
    @Column(name = "profile_image_content_type", length = 60)
    private String contentType;

    public ProfileImage(String storedName, String contentType) {
        if (storedName == null || storedName.isBlank()) {
            throw new IllegalArgumentException("저장된 파일 이름이 필요합니다.");
        }
        ...
    }
}
```

### 2. 저장 방식 — 별도 테이블 없음

- `@Embeddable` 필드는 **주인 엔티티 테이블(member)의 컬럼**으로 펼쳐진다. `notification_preference` 테이블은 생기지 않는다. `@Id` 도 없다.
- `profile_image_` 접두어는 같은 member 테이블 안에서 컬럼 이름이 겹치지 않게 하려는 것.
- **DB 가 똑같다 → 묶어서 얻는 건 전부 객체 쪽**에 있다.

### 3. 묶어서 얻는 것 두 가지

> **① 응집도** — 데이터와 그 데이터를 다루는 행동(`NotificationPreference.remindsAt(now, startAt)`)을 한 클래스에. 묶지 않으면 로직이 이미 필드가 많은 `Member` 에 들어가 구분이 어려워진다.
>
> **② 불변식(invariant) 보장** — 생성자에서 검증하니 "파일 이름은 비었는데 형식만 있는" 잘못된 조합이 **존재할 수 없다**. 필드 두 개로 흩어져 있으면 막을 곳이 없다.

### 4. 공유하면 생기는 문제 — 앨리어싱

setter 가 있었다고 가정:

```java
@Transactional
void copyPreference(Member memberA, Member memberB) {   // 둘 다 영속
    NotificationPreference pref = memberA.getNotificationPreference();
    memberB.setNotificationPreference(pref);   // 같은 인스턴스를 넘김
    pref.setCommentEnabled(false);             // A 만 끄려고 했다
}
```

```
memberA.notificationPreference ──┐
                                 ├──▶ [ 객체 #1 ]  commentEnabled = false
memberB.notificationPreference ──┘
```

- `memberB.getNotificationPreference().isCommentEnabled()` → **false** (같은 객체).
- 변경 감지는 **엔티티별로** 스냅샷과 비교 → A, B 둘 다 달라짐 → **UPDATE 2개**. 에러 없이 B 의 설정까지 꺼진다.
- 엔티티는 id 로 구분되니 공유돼도 "같은 엔티티"지만, **값 타입은 공유되는 순간 남의 값을 바꾸는 부작용**.

### 5. 막는 방법 — 불변 + 교체

```java
// Member.java — 실제 코드
this.notificationPreference = notificationPreference.change(reminderLeadMinutes, toggles);
// NotificationPreference 에는 setter 가 없고 change() 는 새 객체를 반환
```

```
변경 전:   memberA ──┐
                     ├──▶ [ 객체 #1 ] commentEnabled = true
           memberB ──┘
변경 후:   memberA ──▶ [ 객체 #2 ]  ← 새 객체
           memberB ──▶ [ 객체 #1 ] commentEnabled = true (그대로)
```

> setter 가 없고 변경 시 새 객체로 **교체**하면, 아무도 공유된 객체 자체를 바꿀 수 없어서 한쪽의 변경이 다른 참조자에게 번지지 않는다. (`String` 이 불변이라 마음껏 공유되는 원리)

- `ProfileImage` 도 `this.profileImage = newImage` 로 `storedName` + `contentType` 을 **통째로** 교체 → 경로만 바뀌고 형식은 예전 것인 어긋난 상태가 안 생긴다.

## 오답·헷갈린 점

- **전**: 묶어서 얻는 것 = "엔티티 구조와 서비스 로직을 간략히 할 수 있다" → **후**: 이름을 붙일 것 — **응집도** + **불변식 보장**.
- **전**: 공유된 값 타입을 setter 로 고치면 "A 만 바뀐다" → **후**: 같은 객체를 가리키므로 **B 도 바뀐다** → UPDATE **2개**.
- **전**: 그때 B 에서 읽으면 "true" → **후**: **false**.
- **전**: 두 엔티티 모두 스냅샷과 "같다, UPDATE 1개" → **후**: 둘 다 다르다 → **2개**.
- **전**: 그 현상의 이름 = "영속성 전이" → **후**: **앨리어싱(aliasing)**. JPA 와 무관한 자바 참조 문제. 영속성 전이는 cascade.
  - ⚠️ 10/10 `shallow-immutability-defensive-copy` 에서 배운 개념인데 연결하지 못함. 힌트 후에도 3개 모두 오답 → 메모리 화살표 그림으로 다시 연습할 것.
- **전**: 프로필 이미지는 "이미지 경로만 계속 업데이트" → **후**: setter 없음. 새 객체로 **통째로 교체**.
- **전**: 최종 답변에서 묶는 이유 = "**불변성** 보장과 응집도" → **후**: 묶는 이유는 **불변식**(invariant, 생성자 검증 규칙). **불변성**(immutability, 값이 안 바뀜)은 공유 문제의 해법. 비슷하게 들리지만 다른 개념.

| | 뜻 | 예시 |
|---|---|---|
| 불변식 (invariant) | 객체가 항상 지켜야 하는 규칙 | ProfileImage 생성자가 빈 이름 거부 |
| 불변성 (immutability) | 만든 뒤 값이 바뀌지 않음 | setter 없이 change() 가 새 객체 반환 |

## Q&A

- **Q. NotificationPreference 의 필드는 DB 어디에?** A. member 테이블의 컬럼. 별도 테이블 없음.
- **Q. DB 가 같은데 묶는 이유?** A. 응집도, 불변식 보장.
- **Q. 같은 값 타입 인스턴스를 두 엔티티가 공유하고 setter 로 고치면?** A. 앨리어싱 → 둘 다 바뀌고 UPDATE 2개.
- **Q. study-board 는 어떻게 막나?** A. setter 없이 change() 가 새 객체를 반환해 교체.

## 다음 학습

- 값 타입의 equals/hashCode 는 **모든 필드**로 비교해야 하는 이유 (엔티티는 id) → [[jpa-entity-equality]]
- Java `record` 를 `@Embeddable` 로 쓸 수 있나 (Hibernate 6.2+)
- 연결: [[shallow-immutability-defensive-copy]], [[string-immutability]], [[dirty-checking]], [[2026-10-18-value-type-vs-entity]]
