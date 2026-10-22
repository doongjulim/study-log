---
type: refined
slug: unboxing-npe-wrapper-null
tags: [오토언박싱, auto-unboxing, 언박싱, unboxing, 오토박싱, autoboxing, intValue, NullPointerException, NPE, 언박싱NPE, 런타임예외, runtime-exception, 컴파일에러, compile-error, 컴파일러는타입만본다, Map.get, getOrDefault, 래퍼타입, wrapper-type, Integer, Long, int, null의미, DTO, 요청DTO, request-dto, 부분수정, PATCH, 미전송, absent-field, Jackson, NotNull, Bean-Validation, 400, 면접답변]
topic: "래퍼 타입의 null — 자동 언박싱 NPE 는 실행 중에 터지고, DTO 에 Integer 를 쓰면 '안 보냄'을 구별하는 대신 null 처리 책임이 생긴다"
summary: "int c = map.get(키) 는 타입상 Integer→int 가 합법이라 컴파일은 되지만, 컴파일러가 intValue() 호출로 바꿔 쓰므로 null 이 오면 실행 중 NullPointerException 이 난다. 컴파일러는 타입만 보고 값은 모른다. 요청 DTO 필드를 int 로 두면 미전송이 0 이 되어 0 으로 바꾸려는 요청과 구별할 수 없고, Integer 로 두면 null 로 구별되지만 그 null 을 int 로 바꾸는 순간 같은 NPE 가 나므로 부분 수정은 null 체크, 필수 값은 @NotNull 로 명시 처리한다."
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-5.html#jls-5.1.8
  - https://docs.spring.io/spring-framework/reference/core/validation/beanvalidation.html
updated: 2026-10-22
---

# 래퍼 타입의 null — 언박싱 NPE 와 DTO 필드 선택

## 개념 정의 — 언박싱은 메서드 호출이다

```java
Map<String, Integer> stock = new HashMap<>();
stock.put("banana", 5);

int count = stock.get("apple");
// 컴파일러가 실제로 바꿔 쓰는 코드
int count = stock.get("apple").intValue();   // null.intValue() → NullPointerException
```

- `Map.get` 은 키가 없으면 `null` 을 돌려준다. `int` 는 null 을 담을 수 없다 ([[primitive-vs-reference-type]]).
- 래퍼를 기본형에 넣는 **자동 언박싱**은 `intValue()` / `longValue()` 호출로 컴파일된다. 대상이 null 이면 그 호출에서 NPE 가 난다.
- 대안: `stock.getOrDefault("apple", 0)`, 또는 `Integer` 로 받아 null 을 확인한다.

## 작동 원리 — 왜 컴파일러가 못 잡나

| 코드 | 컴파일러가 알 수 있나? | 결과 |
|---|---|---|
| `int x; System.out.println(x);` | ⭕ 코드만 봐도 "값을 넣지 않고 읽는다"가 보임 | **컴파일 에러** |
| `int c = stock.get("apple");` | ❌ `Integer → int` 는 타입상 합법. 맵에 무엇이 있을지는 실행해 봐야 앎 | **런타임 NPE** |

**컴파일러는 타입만 검사하고 값은 모른다.** 같은 "값 없음" 문제가 하나는 컴파일 에러, 하나는 런타임 NPE 가 되는 이유다.

## 트레이드오프 — DTO 필드는 `int` 인가 `Integer` 인가

```java
public class MemberUpdateRequest {
    private int age;       // A안
    // private Integer age; // B안
}
// 요청 JSON: {"name": "DJ"}   ← age 를 아예 안 보냄
```

| | A안 `int` | B안 `Integer` |
|---|---|---|
| 미전송 시 값 | 0 (필드 기본값) | null |
| "0으로 수정" vs "수정 안 함" 구별 | ❌ 둘 다 0 | ⭕ 0 vs null |
| 대가 | 의미 손실 | **null 처리 책임** — 언박싱하는 순간 NPE |

B안을 택했을 때 NPE 가 터지는 지점:

```java
member.setAge(request.getAge());   // 엔티티 필드가 int → 언박싱 NPE
if (request.getAge() > 0) { ... }  // 비교하려고 언박싱 → NPE
```

null 을 명시적으로 처리한다:
- **부분 수정**: `if (request.getAge() != null) member.changeAge(request.getAge());` — 보낸 값만 반영한다.
- **필수 값**: `@NotNull` 로 요청 단계에서 400 을 돌려준다. 원인 모를 500 보다 낫다 ([[exception-status-code-mapping]]).

엔티티 `@Id` 를 `Long` 으로 두는 것도 같은 축이다. "아직 채번 전"을 null 로 표현하기 위해서다 ([[wrapper-cache-autoboxing]]). 래퍼를 택하면 null 표현력을 얻는 대신 `==` 캐시 함정과 언박싱 NPE 를 함께 떠안는다.

> [!WARNING]
> **오답 코너**
> - **"`int count = map.get(없는 키)` 는 컴파일 에러"** — 컴파일은 된다. 실행 중 **언박싱 NPE** 가 난다. 원인(get 이 null 을 돌려주고 int 는 null 불가)은 맞아도 **시점**을 틀리기 쉽다.
> - **"`Integer` 로 바꾸면 해결"** — 구별은 가능해지지만, null 을 `int` 로 바꾸는 순간 같은 NPE 가 난다. null 체크나 `@NotNull` 이 함께 있어야 한다.

## 면접 답변

> "래퍼 타입을 기본형에 대입하면 컴파일러가 intValue() 같은 언박싱 호출을 넣는데, 값이 null 이면 실행 중에 NullPointerException 이 납니다. 컴파일러는 타입만 검사하기 때문에 미리 잡지 못합니다. 그래서 요청 DTO 에서 값을 안 보낸 것과 0 을 구별해야 할 때는 Integer 를 쓰되, null 체크나 @NotNull 검증으로 null 을 명시적으로 처리합니다."

## 복습 체크

- [ ] `int count = map.get("없는키")` 는 언제 어떤 에러가 나는가?
- [ ] 자동 언박싱은 어떤 메서드 호출로 컴파일되는가?
- [ ] 초기화 안 된 지역변수는 컴파일 에러인데 언박싱 NPE 는 왜 런타임에 나는가?
- [ ] 요청 DTO 숫자 필드를 `int` 와 `Integer` 로 둘 때 각각 무엇을 얻고 잃는가?
- [ ] `Integer` 필드를 택했을 때 NPE 가 나는 지점과 대응 방법 두 가지는?

## 관련

[[primitive-vs-reference-type]] · [[wrapper-cache-autoboxing]] · [[exception-status-code-mapping]] · [[java-optional]]
