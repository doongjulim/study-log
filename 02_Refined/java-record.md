---
type: refined
slug: java-record
tags: [record, 레코드, Java16, Java17, 레코드클래스, record-class, DTO, 요청DTO, 응답DTO, 불변DTO, immutable, final, 정규생성자, canonical-constructor, 컴팩트생성자, compact-constructor, 접근자, accessor, JavaBeans, getter, setter, Lombok, Jackson, 역직렬화, deserialization, RequestBody, ModelAttribute, 생성자바인딩, constructor-binding, 리플렉션, getRecordComponents, parameters옵션]
topic: record 가 무엇을 자동으로 만들어 주고, 왜 요청/응답 DTO 에 잘 맞는가
summary: record 는 필드가 private final 이고 setter 가 없어 요청 데이터가 계층을 지나는 동안 바뀌지 않는다. 컴파일러가 정규 생성자·이름 그대로의 접근자(title())·equals/hashCode/toString 을 만들고, 컴포넌트 이름이 클래스 파일에 남아 Jackson 이 setter 대신 정규 생성자로 값을 주입할 수 있다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/en/java/javase/17/language/records.html
  - https://openjdk.org/jeps/395
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Class.html#getRecordComponents()
updated: 2026-09-26
---

# record

## 개념 정의

값을 담아 나르는 것이 목적인 클래스를 한 줄로 선언하는 문법 (Java 16 정식).

```java
record CreatePostRequest(String title, String content) {}
```

**왜 DTO 에 쓰는가.** 요청 DTO 는 컨트롤러 → 검증 → 서비스 → 다른 메서드로 흘러간다.
일반 클래스 + Lombok `@Getter @Setter` 라면 그 중간 어디서든 `setTitle(...)` 을 호출할 수 있어
**처음 받은 요청 데이터가 변조**될 수 있다. record 는 필드가 `private final` 이고 setter 가 없어서
한 번 만들어진 값이 계층을 지나는 동안 바뀌지 않는다.

## 작동 원리

### 컴파일러가 만들어 주는 것

| 구분 | 일반 클래스 DTO (+Lombok) | record |
|---|---|---|
| 필드 | 보통 가변 (setter 있음) | `private final` 자동 |
| 생성 | 기본 생성자로 빈 객체 → setter 로 채움 | **정규 생성자(canonical constructor)** 로 한 번에 |
| 접근자 | `getTitle()` (JavaBeans 규약) | **`title()`** — 컴포넌트 이름 그대로 |
| 그 외 | Lombok 필요 | `equals`/`hashCode`/`toString` 자동 (전체 컴포넌트 기반) |

정규 생성자 = 모든 컴포넌트를 선언 순서대로 받는 생성자.

### 기본 생성자도 setter 도 없는데 요청 값은 어떻게 들어가나

**setter 주입이 아니라 정규 생성자 주입**이다. 객체를 만드는 순간 JSON 키와 이름이 같은 파라미터에 값을 매칭한다.

- 일반 클래스는 `.class` 파일에 **생성자 파라미터 이름이 기본적으로 남지 않는다** (`-parameters` 컴파일 옵션을 켜야 남음).
  그래서 예전엔 `@JsonCreator` + `@JsonProperty` 로 "이 키는 몇 번째 파라미터"라고 알려줘야 했다.
- record 는 **컴포넌트 이름과 순서가 클래스 파일 메타데이터(Record 속성)에 남는다.**
  Jackson(2.12+) 은 `Class#getRecordComponents()` 로 이를 읽어 JSON 키 ↔ 정규 생성자 파라미터를 매칭한다.

### 컴팩트 생성자

파라미터 목록을 생략하고 **검증·가공 로직만** 넣는 record 전용 생성자. 안에서 파라미터를 재할당하면 그 값이 필드에 대입된다.

```java
record CreatePostRequest(String title, List<String> tags) {
    CreatePostRequest {
        if (title == null || title.isBlank()) throw new IllegalArgumentException("title");
        tags = List.copyOf(tags);   // 방어적 복사 → [[shallow-immutability-defensive-copy]]
    }
}
```

## 트레이드오프 / 한계

- **불변성은 얕다.** `final` 은 참조만 고정하므로 `List` 같은 가변 객체를 담으면 내용은 바뀐다 → [[shallow-immutability-defensive-copy]].
- `equals`/`hashCode` 가 **전체 컴포넌트 기반**이라, 식별자 기반 동등성이 필요한 JPA 엔티티에는 맞지 않는다 ([[jpa-entity-equality]]).
  엔티티는 기본 생성자·프록시 상속·변경 감지가 필요한데 record 는 `final` 클래스이고 필드도 `final` 이라 애초에 엔티티로 쓸 수 없다.
- 다른 클래스를 상속할 수 없다(암묵적으로 `java.lang.Record` 상속). 인터페이스 구현은 가능.

> [!WARNING]
> **오답 코너**
> - **"record 접근자는 `getTitle()`"** — **`title()`**. record 는 JavaBeans 규약을 따르지 않는다.
> - **"Jackson 이 setter 로 값을 넣는다"** — setter 가 없다. **정규 생성자 주입**이며, 컴포넌트 이름이 클래스 파일에 남기 때문에 가능하다.
> - **"record 면 완전히 불변"** — 참조만 고정되는 **얕은 불변**이다.

## 복습 체크

- [ ] 요청 DTO 를 record 로 만들면 일반 클래스 + setter 대비 무엇을 막을 수 있는가?
- [ ] `record R(String title, String content)` 를 컴파일하면 생기는 멤버를 모두 말할 수 있는가?
- [ ] record 의 접근자 호출 형태는?
- [ ] 기본 생성자도 setter 도 없는 record 에 Jackson 이 값을 넣는 방법과, 일반 클래스에선 왜 그게 어려웠는지 설명할 수 있는가?
- [ ] 컴팩트 생성자는 무엇이고 어디에 쓰는가?
- [ ] record 를 JPA 엔티티로 쓸 수 없는 이유는?

## 관련

[[shallow-immutability-defensive-copy]] · [[jpa-entity-equality]] · [[equals-hashcode-contract]] · [[string-immutability]]
