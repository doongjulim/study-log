---
date: 2026-09-26
type: raw
author: dongju
tags: [record, 레코드, Java 16, Java 17, DTO, 요청 DTO, 응답 DTO, 불변, immutable, 불변객체, final, 얕은 불변, shallow immutability, 정규 생성자, canonical constructor, 컴팩트 생성자, compact constructor, 접근자, accessor, JavaBeans, getter, setter, Lombok, Jackson, 역직렬화, deserialization, "@RequestBody", "@ModelAttribute", 리플렉션, reflection, getRecordComponents, -parameters, 방어적 복사, defensive copy, List.copyOf, Collections.unmodifiableList, unmodifiable view, 뷰 vs 복사, UnsupportedOperationException, 참조, reference]
topic: record 를 DTO 로 쓰는 이유와, record 의 불변성이 실제로 어디까지 보장되는가
summary: record 는 필드가 private final 이고 setter 가 없어 요청 DTO 가 계층을 지나는 동안 값이 바뀌지 않게 해 준다. 값은 setter 가 아니라 정규 생성자로 주입되며(Jackson 은 클래스 파일에 남은 컴포넌트 이름으로 매칭), final 은 참조만 고정하는 얕은 불변이라 컬렉션 필드는 컴팩트 생성자에서 List.copyOf 로 방어적 복사를 해야 완전히 막힌다.
source: session
distilled: true
distilled_at: 2026-09-26
---

## 배운 개념

### 1. 왜 DTO 를 record 로 만드는가 (What & Why)

study-board(Java 17, Spring Boot 3.3) 에서 **요청/응답 DTO** 를 record 로 만들었다.

일반 클래스 + Lombok `@Getter @Setter` DTO 의 문제:
요청 DTO 는 컨트롤러 → 검증 → 서비스 → 다른 메서드로 흘러간다. 그 중간 어디서든
`setTitle(...)` 을 호출할 수 있으면 **처음 받은 요청 데이터가 변조되어 무결성이 깨진다.**

record 는 필드에 `private final` 이 자동으로 붙고 setter 가 없다.
→ 한 번 만들어진 요청 데이터는 계층을 지나는 동안 바뀌지 않는다.

### 2. 컴파일러가 만들어 주는 것 (How)

```java
record CreatePostRequest(String title, String content) {}
```

| 구분 | 일반 클래스 DTO (+Lombok) | record DTO |
|---|---|---|
| 필드 | 보통 가변(setter 있음) | `private final` 자동 |
| 생성 | 기본 생성자로 빈 객체 → setter 로 채움 | **정규 생성자(canonical constructor)** 로 한 번에 생성 |
| 접근자 | `getTitle()` (JavaBeans 규약) | **`title()`** (컴포넌트 이름 그대로) |
| 자동 생성 | Lombok 필요 | 정규 생성자, 접근자, `equals`/`hashCode`/`toString` |

- 정규 생성자 = 모든 컴포넌트를 순서대로 받는 생성자. 위 예시에서는 파라미터 2개.

### 3. 기본 생성자도 setter 도 없는데 요청 값은 어떻게 들어가나

**setter 주입이 아니라 정규 생성자 주입**이다. 객체를 만드는 순간에 JSON 키와 이름이 같은 파라미터에 값을 매칭한다.

이게 가능한 이유:

- 일반 클래스는 `.class` 파일에 **생성자 파라미터 이름이 기본적으로 남지 않는다**(`-parameters` 컴파일 옵션을 켜야 남음).
  그래서 예전에는 `@JsonCreator` + `@JsonProperty` 로 "이 JSON 키는 몇 번째 파라미터"라고 알려줘야 했다.
- record 는 **컴포넌트 이름과 순서가 클래스 파일 메타데이터(Record 속성)에 남는다.**
  Jackson(2.12+) 은 리플렉션 `Class#getRecordComponents()` 로 이를 읽어 JSON 키 ↔ 정규 생성자 파라미터를 매칭한다.

### 4. final 은 참조만 고정한다 — 얕은 불변 (Edge case)

```java
record CreatePostRequest(String title, List<String> tags) {}

List<String> list = new ArrayList<>(List.of("java"));   // list = ["java"]
CreatePostRequest req = new CreatePostRequest("제목", list);
// req.tags 와 list 는 "같은 주소"를 가리킨다 (record 는 복사하지 않고 주소만 저장)

list.add("spring");     // 같은 객체에 추가 → ["java", "spring"]
req.tags().add("jpa");  // 같은 객체에 추가 → ["java", "spring", "jpa"]
```

```java
req.tags = new ArrayList<>();  // A: 컴파일 에러 — final 이 참조 재할당을 막음
req.tags().add("jpa");         // B: 실행됨 — 참조가 가리키는 객체의 내용은 못 막음
```

→ `final` 이 지키는 것은 **참조(주소)** 이지, **참조가 가리키는 객체의 내용**이 아니다.
이를 **얕은 불변(shallow immutability)** 이라 한다.

### 5. 읽기 전용 리스트: view vs copy

| | `Collections.unmodifiableList(list)` | `List.copyOf(list)` |
|---|---|---|
| 정체 | 원본을 감싼 **view** (원본 주소 보유) | 새로 만든 **copy** |
| `add()` 호출 | `UnsupportedOperationException` | `UnsupportedOperationException` |
| 원본이 바뀌면 | **그대로 보인다** | 영향 없음 |
| 비유 | 남의 집 CCTV — 내가 가구는 못 옮기지만 집주인이 옮기면 화면에 보임 | 집 사진 — 집주인이 뭘 해도 사진은 그대로 |

```java
List<String> list = new ArrayList<>(List.of("java"));
List<String> a = Collections.unmodifiableList(list);
List<String> b = List.copyOf(list);
list.add("spring");
// a → ["java", "spring"]   (view 라서 원본 변경이 보임)
// b → ["java"]             (copy 라서 그대로)
```

### 6. 해결: 컴팩트 생성자 + 방어적 복사

```java
record CreatePostRequest(String title, List<String> tags) {
    CreatePostRequest {             // 컴팩트 생성자: 파라미터 목록 생략, 검증/가공만
        tags = List.copyOf(tags);   // 방어적 복사 + 수정 불가
    }
}
```

- 컴팩트 생성자 안에서 파라미터를 재할당하면, 그 값이 필드에 대입된다.
- 외부 `list.add()` (①) 와 `req.tags().add()` (②) 를 **둘 다** 막는다.
- 주의: `List.copyOf` 는 리스트 자체나 요소가 `null` 이면 `NullPointerException`.
  태그가 없을 수 있는 요청이면 `tags = (tags == null) ? List.of() : List.copyOf(tags);`

## 오답·헷갈린 점

- **접근자 이름**
  - 전: record 접근자는 `getTitle()` 이다.
  - 후: **`title()`**. record 는 JavaBeans 규약을 따르지 않고 컴포넌트 이름을 그대로 쓴다.
- **공유 리스트 결과 추적**
  - 전: ①② 실행 후 `req.tags()` 는 `["spring", "jpa"]`.
  - 후: **`["java", "spring", "jpa"]`**. 처음 넣은 `"java"` 는 사라지지 않는다. record 는 주소만 저장하므로 바깥 `list` 와 `req.tags` 는 같은 객체이고, 어느 쪽에서 추가해도 한 리스트에 쌓인다.
- **final 이 보장하는 것**
  - 전: "객체 생성 시점의 무결성을 보장한다."
  - 후: 생성 이후에도 계속 **참조(주소) 재할당**을 막는다. 다만 참조가 가리키는 객체의 **내용**은 막지 못한다 → 얕은 불변.
- **`readonly` 키워드**
  - 전: "readonly 를 적용한다."
  - 후: Java 에는 `readonly` 키워드가 없다(C#/TypeScript). `Collections.unmodifiableList` 또는 `List.copyOf` 를 쓴다.
- **view vs copy — 이유는 맞고 결론이 반대였음**
  - 전: "A(unmodifiableList)만 외부 수정을 막는다. 원본 주소를 가지고 있기 때문에."
  - 후: 원본 주소를 가지고 있기 **때문에** A 는 원본 변경이 **그대로 보인다**. 외부 수정까지 막는 건 복사본을 만드는 **B(`List.copyOf`)**.

## Q&A

- **Q. record 를 어디에 썼고, 일반 클래스와 뭐가 다른가?**
  A. 요청/응답 DTO. 필드가 자동으로 `private final` 이고 setter 가 없어서 요청 데이터가 계층을 지나는 동안 변조되지 않는다. 정규 생성자, `title()` 형태의 접근자, `equals`/`hashCode`/`toString` 을 컴파일러가 만들어 준다.
- **Q. 기본 생성자도 setter 도 없는데 Jackson 은 어떻게 값을 넣나?**
  A. 정규 생성자 주입. record 는 컴포넌트 이름이 클래스 파일에 남아 있어서, Jackson 이 리플렉션(`getRecordComponents()`)으로 읽고 JSON 키를 생성자 파라미터에 매칭한다.
- **Q. `record R(List<String> tags)` 는 완전히 불변인가?**
  A. 아니다. `final` 은 참조만 고정하는 얕은 불변이다. 외부에서 원본 리스트를 바꾸거나 `r.tags().add()` 를 하면 내용이 바뀐다.
- **Q. 어떻게 막나?**
  A. 컴팩트 생성자에서 `tags = List.copyOf(tags);`. `unmodifiableList` 는 view 라 원본 변경이 보이므로 부족하다.
- **30초 면접 답변**
  > "study-board 에서 요청/응답 DTO 를 record 로 만들었습니다. 필드가 자동으로 final 이고 setter 가 없어서, 요청 데이터가 계층을 지나는 동안 바뀌지 않습니다. Jackson 은 정규 생성자로 값을 바인딩하기 때문에 기본 생성자 없이도 동작합니다. 다만 record 는 참조만 고정하는 얕은 불변이라, List 같은 컬렉션 필드는 컴팩트 생성자에서 List.copyOf 로 방어적 복사를 해서 외부 수정까지 막았습니다."

## 다음 학습

- record 가 자동 생성하는 `equals`/`hashCode` 의 동작과 한계 — 전체 필드 기반이라 JPA 엔티티에 부적합한 이유 (2026-09-10 equals 노트에서 이어짐)
- record 를 JPA 엔티티로 못 쓰는 이유 — 기본 생성자·프록시(상속)·final 필드와 변경 감지(dirty checking)의 충돌
- `@ModelAttribute`(Thymeleaf 폼 바인딩) 에서 record 생성자 바인딩이 동작하는 방식 (Spring 6)
- 컴팩트 생성자에서의 검증 로직 vs Bean Validation(`@NotBlank` 등) 역할 분담
