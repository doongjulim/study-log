---
type: refined
slug: shallow-immutability-defensive-copy
tags: [얕은불변, shallow-immutability, 깊은불변, deep-immutability, final, 참조, reference, 참조재할당, 방어적복사, defensive-copy, List.copyOf, Collections.unmodifiableList, unmodifiable, 읽기전용, read-only, readonly, 뷰, view, 복사본, copy, UnsupportedOperationException, 컴팩트생성자, compact-constructor, record, 불변객체, 가변컬렉션, aliasing, 앨리어싱]
topic: final 이 실제로 지켜 주는 것(참조)과 지켜 주지 않는 것(내용), 그리고 view 가 아닌 copy 로 막아야 하는 이유
summary: final 은 참조 재할당만 막고 참조가 가리키는 객체의 내용은 막지 못한다(얕은 불변). 컬렉션을 받아 저장하면 외부와 같은 객체를 공유하게 되므로, 수정 불가 view(unmodifiableList)가 아니라 복사본(List.copyOf)으로 저장해야 외부 수정까지 차단된다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/List.html#copyOf(java.util.Collection)
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Collections.html#unmodifiableList(java.util.List)
updated: 2026-09-26
---

# 얕은 불변과 방어적 복사

## 개념 정의

**`final` 이 지키는 것은 참조(주소)이지, 참조가 가리키는 객체의 내용이 아니다.**
필드가 전부 `final` 이어도 그 안에 가변 객체가 있으면 내용은 바뀐다. 이를 **얕은 불변(shallow immutability)** 이라 한다.

```java
record CreatePostRequest(String title, List<String> tags) {}

req.tags = new ArrayList<>();  // 컴파일 에러 — final 이 참조 재할당을 막음
req.tags().add("jpa");         // 실행됨 — 참조가 가리키는 객체의 내용은 못 막음
```

## 작동 원리

### 저장할 때 복사하지 않으면 "같은 객체"를 공유한다

```java
List<String> list = new ArrayList<>(List.of("java"));   // ["java"]
CreatePostRequest req = new CreatePostRequest("제목", list);
// req.tags 와 list 는 같은 주소 — 생성자는 복사하지 않고 주소만 저장한다

list.add("spring");     // 같은 객체에 추가 → ["java", "spring"]
req.tags().add("jpa");  // 같은 객체에 추가 → ["java", "spring", "jpa"]
```

두 경로 모두 막아야 한다: ① 외부에서 원본을 수정 ② 접근자로 꺼내서 수정.

### view vs copy

| | `Collections.unmodifiableList(list)` | `List.copyOf(list)` |
|---|---|---|
| 정체 | 원본을 감싼 **view** (원본 주소 보유) | 새로 만든 **copy** |
| ② `add()` 호출 | `UnsupportedOperationException` | `UnsupportedOperationException` |
| ① 원본이 바뀌면 | **그대로 보인다** | 영향 없음 |
| 비유 | 남의 집 **CCTV** — 내가 가구는 못 옮기지만 집주인이 옮기면 화면에 보인다 | 집 **사진** — 집주인이 뭘 해도 그대로 |

```java
List<String> a = Collections.unmodifiableList(list);
List<String> b = List.copyOf(list);
list.add("spring");
// a → ["java", "spring"]   (view)
// b → ["java"]             (copy)
```

### 해결: 받을 때 복사한다

```java
record CreatePostRequest(String title, List<String> tags) {
    CreatePostRequest {
        tags = List.copyOf(tags);   // 방어적 복사 + 수정 불가 → ①② 모두 차단
    }
}
```

## 트레이드오프 / 한계

- `List.copyOf` 는 리스트 자체나 **요소가 `null` 이면 `NullPointerException`**. 없을 수 있는 입력이면
  `tags = (tags == null) ? List.of() : List.copyOf(tags);`
- 복사 비용(O(n))이 든다. 큰 컬렉션을 자주 생성하는 경로라면 고려 대상.
- **여전히 한 단계만 막는다.** 요소 자체가 가변 객체(`List<Member>`)라면 요소 내부는 바뀔 수 있다 — 깊은 불변은 요소 타입까지 불변이어야 성립한다.
  `String` 이 안전한 요소인 이유는 그 자체가 불변이기 때문이다 ([[string-immutability]]).
- `List.copyOf` 는 입력이 이미 불변 리스트면 복사 없이 그대로 반환할 수 있다(구현 최적화).

> [!WARNING]
> **오답 코너**
> - **"final 은 객체 생성 시점의 무결성을 보장한다"** — 생성 이후에도 계속 **참조 재할당**을 막는다. 다만 **내용**은 못 막는다.
> - **"①② 후 결과는 `["spring", "jpa"]`"** — **`["java", "spring", "jpa"]`**. 처음 넣은 값은 그대로이고, 같은 객체에 계속 쌓인다.
> - **"readonly 를 적용한다"** — Java 에 `readonly` 키워드는 없다(C#/TypeScript). `unmodifiableList` 또는 `List.copyOf`.
> - **"unmodifiableList 는 원본 주소를 가지므로 외부 수정을 막는다"** — 이유는 맞고 결론이 반대. 원본 주소를 가지니까 **원본 변경이 그대로 보인다.** 외부 수정까지 막는 건 `List.copyOf`.

## 복습 체크

- [ ] `final` 이 막는 것과 못 막는 것을 코드 두 줄로 보여줄 수 있는가?
- [ ] 위 예제에서 `list.add("spring"); req.tags().add("jpa");` 후 `req.tags()` 의 값은?
- [ ] `unmodifiableList` 와 `List.copyOf` 중 외부 원본 수정까지 막는 것은 무엇이고 왜인가? (CCTV vs 사진)
- [ ] record 에서 방어적 복사를 넣는 위치와 코드는?
- [ ] `List.copyOf` 의 null 동작과 대응 방법은?

## 관련

[[java-record]] · [[string-immutability]] · [[reference-vs-value-equality]] · [[equals-hashcode-contract]]
