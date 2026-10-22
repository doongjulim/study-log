---
date: 2026-10-22
type: raw
author: dongju
tags: [오토언박싱, auto-unboxing, 언박싱, unboxing, 오토박싱, autoboxing, intValue, NullPointerException, NPE, 런타임예외, runtime-exception, 컴파일에러, compile-error, 컴파일러는타입만, Map.get, getOrDefault, 래퍼타입, wrapper-type, Integer, int, DTO, 요청DTO, request-dto, 부분수정, PATCH, 미전송, absent-field, null의미, Jackson, NotNull, Bean-Validation, 400, 면접답변]
topic: 래퍼 타입의 null — 자동 언박싱 NPE 는 컴파일이 아니라 실행 중에 터지고, DTO 에 Integer 를 쓰면 "안 보냄"을 구별하는 대신 null 처리 책임이 생긴다
summary: int c = map.get("없는키") 는 타입상 Integer→int 가 합법이라 컴파일은 되지만, 컴파일러가 intValue() 호출로 바꿔 쓰므로 null 이 오면 실행 중 NullPointerException 이 난다. 컴파일러는 타입만 보고 값은 모르기 때문이다. 요청 DTO 필드를 int 로 두면 미전송이 0 이 되어 0 으로 바꾸려는 요청과 구별할 수 없고, Integer 로 두면 null 로 구별되지만 그 null 을 int 로 바꾸는 순간 같은 NPE 가 나므로 부분 수정은 null 체크, 필수 값은 @NotNull 로 명시 처리한다.
source: session
distilled: true
distilled_at: 2026-10-22
---

## 배운 개념

### 1. 자동 언박싱 NPE — 컴파일은 되고, 실행 중에 터진다

```java
Map<String, Integer> stock = new HashMap<>();
stock.put("banana", 5);

int count = stock.get("apple");
// 컴파일러가 실제로 바꿔 쓰는 코드
int count = stock.get("apple").intValue();   // null.intValue() → NullPointerException
```

- `Map.get` 은 키가 없으면 `null` 을 돌려준다. `int` 는 null 을 담을 수 없다 → 언박싱 시점에 NPE.
- 대안: `stock.getOrDefault("apple", 0)`, 또는 `Integer` 로 받아 null 체크.

### 2. 왜 컴파일러가 못 잡나 — 타입은 알고 값은 모른다

| 코드 | 컴파일러가 알 수 있나? | 결과 |
|---|---|---|
| `int x; System.out.println(x);` | ⭕ 코드만 봐도 "값을 넣지 않고 읽는다" | **컴파일 에러** |
| `int c = stock.get("apple");` | ❌ `Integer → int` 는 타입상 합법. 맵에 뭐가 있을지는 실행해 봐야 앎 | **런타임 NPE** |

### 3. DTO 필드 — `int` vs `Integer`

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
| 대가 | 의미 손실 | null 처리 책임 — 언박싱하는 순간 NPE |

B안을 택했을 때 터지는 지점:

```java
member.setAge(request.getAge());   // 엔티티 필드가 int → 언박싱 NPE
if (request.getAge() > 0) { ... }  // 비교하려고 언박싱 → NPE
```

명시적 처리:
- **부분 수정**: `if (request.getAge() != null) member.changeAge(request.getAge());` — 보낸 값만 반영.
- **필수 값**: `@NotNull` 로 요청 단계에서 400 → 원인 모를 500 보다 낫다 ([[exception-status-code-mapping]]).
- (엔티티 `@Id` 가 `Long` 인 이유도 같은 축 — "아직 없음"을 null 로 표현. [[wrapper-cache-autoboxing]])

### 면접 답변

> "래퍼 타입을 기본형에 대입하면 컴파일러가 intValue() 같은 언박싱 호출을 넣는데, 값이 null 이면 실행 중에 NullPointerException 이 납니다. 컴파일러는 타입만 검사하기 때문에 미리 잡지 못합니다. 그래서 요청 DTO 에서 값을 안 보낸 것과 0 을 구별해야 할 때는 Integer 를 쓰되, null 체크나 @NotNull 검증으로 null 을 명시적으로 처리합니다."

## 오답·헷갈린 점

- **전**: `int count = stock.get("apple")` 은 **컴파일 에러** → **후**: 컴파일은 된다. 실행 중 **언박싱 NPE**. 원인 추론("get 이 null 을 돌려주고 int 는 null 불가")은 정확했고 **시점**만 틀림.
- 같은 날 `int x; println(x);` 는 컴파일 에러가 맞았음 → 두 코드의 차이 = **컴파일러가 코드만 보고 알 수 있느냐**(definite assignment vs 런타임 값).
- 👍 DTO A안 0 / B안 null, 구별 가능한 쪽 `Integer` — 정답.
- **누락**: B안을 택했을 때 "무엇을 조심해야 하나"(Q1 과 연결) 답하지 않음 → 인터뷰어가 정리 — 언박싱 NPE, null 체크·`@NotNull`.

## Q&A

**Q. `int count = map.get("없는키")` 는 어떻게 되나?**
A. 컴파일은 되고 실행 중 NPE. 언박싱이 `intValue()` 호출이기 때문.

**Q. 왜 컴파일러가 못 잡나?**
A. `Integer → int` 는 타입상 합법이고, 값이 null 일지는 실행해 봐야 안다. 컴파일러는 타입만 본다.

**Q. 요청 DTO 숫자 필드는 int 인가 Integer 인가?**
A. "안 보냄"과 0 을 구별해야 하면 Integer. 대신 null 을 명시적으로 처리(부분 수정은 null 체크, 필수는 @NotNull).

## 다음 학습

- Jackson 의 `FAIL_ON_NULL_FOR_PRIMITIVES` 와 primitive 필드 역직렬화 동작
- PATCH 부분 수정에서 "null 로 지우기" vs "안 보냄" 구별(`Optional`/`JsonNullable`)
- `Integer` 비교는 `==` 가 아니라 `equals` ([[wrapper-cache-autoboxing]] 복습)
