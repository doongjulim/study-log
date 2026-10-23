---
type: refined
slug: java-array
tags: [배열, array, 자바배열, java-array, 참조형, reference-type, 배열은객체, array-is-object, int배열, "int[]", null가능, new, 앨리어싱, aliasing, 참조복사, 배열복사, Arrays.copyOf, clone, 배열칸, element, 기본값, default-value, length, final필드, 고정크기, fixed-size, equals미재정의, Object.equals, Arrays.equals, toString, Arrays.toString, "[I@", Arrays.hashCode, 배열vsArrayList, array-vs-arraylist, 선택기준, 박싱비용, 면접답변]
topic: "배열은 원소가 int 여도 참조형 객체다 — 대입은 참조를 복사해 앨리어싱이 생기고, length 는 final 필드라 크기가 고정이며, Object 의 equals·toString 을 그대로 쓴다"
summary: "배열 변수에는 힙의 배열 객체를 가리키는 참조가 들어 있어 null 이 가능하고 new 로 만든다. 칸의 내용은 원소 타입이 정해 int[] 칸엔 값 자체, String[] 칸엔 참조가 들어 있고, 객체 안의 칸이라 기본값을 받는다. int[] b = a 는 참조를 복사하므로 b[0] = 99 가 a[0] 도 바꾼다(a == b 는 true). length 는 final 필드라 크기를 바꿀 수 없고, equals·toString·hashCode 를 재정의하지 않아 Arrays.equals·Arrays.toString 을 쓴다. 기본형 대량 데이터·고정 크기에는 박싱이 없는 배열이, 크기가 변하고 API 가 필요한 일반 코드에는 ArrayList 가 맞다."
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-10.html
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Arrays.html
updated: 2026-10-23
---

# 자바 배열

## 개념 정의 — 배열은 참조형 객체, 세 층으로 본다

```
스택                     힙
 a ──┐
     ├──────────▶  int[] 객체 ┌───┬───┬───┐  length = 3 (final 필드)
 b ──┘                        │ 1 │ 2 │ 3 │  ← int[] 칸엔 값 자체
                              └───┴───┴───┘
```

| 층 | 들어 있는 것 |
|---|---|
| 변수 `a`, `b` (스택) | 배열 **객체를 가리키는 참조** → 배열은 참조형, null 가능 |
| `int[]` 객체의 칸 (힙) | `int` **값 자체** |
| `String[]` / `Integer[]` 객체의 칸 (힙) | 원소 객체를 가리키는 **참조** |

- 증거: `int[] x = null;` 은 컴파일된다(기본형은 null 불가). `new int[3]` 처럼 `new` 로 만든다.
- 칸은 **객체 안의 칸**이라 필드처럼 기본값을 받는다 → `new int[3]` 의 칸은 0 ([[variable-kinds-initialization]]).
- "위치는 그릇을 따라가고, 내용은 타입이 정한다" 규칙이 그대로 적용된다 ([[primitive-vs-reference-type]]).

## 작동 원리

### 대입은 참조 복사 → 앨리어싱

```java
int[] a = {1, 2, 3};
int[] b = a;
b[0] = 99;
System.out.println(a[0]);       // 99
System.out.println(a == b);     // true — 같은 객체
```

`int[] b = a` 가 복사하는 "값"은 숫자 1, 2, 3 이 아니라 **변수 a 에 들어 있던 참조**다. `Member m2 = m1` 과 같은 구조다 ([[java-pass-by-value]]). 따로 쓰려면 **새 배열 객체**를 만든다.

```java
int[] c = Arrays.copyOf(a, a.length);   // 또는 a.clone()
c[0] = -1;                              // a[0] 은 그대로, a == c 는 false
```

### `length` 는 `final` 필드 → 크기 고정

```java
int[] arr = new int[3];
arr.length = 5;   // ❌ cannot assign a value to final variable length
```

메서드가 아니라 필드라 괄호가 없다. ArrayList 의 `size()` 는 크기가 변하니 매번 묻는 메서드다. 꽉 찬 배열을 늘리려면 새 배열을 만들어 복사해야 하고, 그 일을 대신 해 주는 것이 ArrayList 다 ([[arraylist-internals]]).

### `Object` 의 메서드를 재정의하지 않는다

```java
int[] x = {1, 2, 3}, y = {1, 2, 3};
x == y                 // false — 다른 객체
x.equals(y)            // false — Object.equals 는 this == obj
Arrays.equals(x, y)    // true

System.out.println(x);                   // [I@1b6d3586 — 타입 이름 + @ + 해시코드 16진수
System.out.println(Arrays.toString(x));  // [1, 2, 3]
System.out.println(List.of(1, 2, 3));    // [1, 2, 3] — List 는 toString 을 재정의
```

| | 배열 | 컬렉션 |
|---|---|---|
| `equals` | 참조 비교 → `Arrays.equals` | 내용 비교 |
| `toString` | `[I@…` → `Arrays.toString` | `[1, 2, 3]` |
| `hashCode` | 객체마다 다름 → `Arrays.hashCode` | 내용 기반 |

배열을 `HashSet`·`HashMap` 키로 쓰면 내용이 같아도 다른 키다 ([[equals-hashcode-contract]]).

## 트레이드오프 — 배열 vs ArrayList

| | 배열 | ArrayList |
|---|---|---|
| 기본형 저장 | `int[]` 값 그대로 (100만 개 약 4MB, 읽기 1번) | `ArrayList<int>` 불가 → `Integer` 박싱 (약 20MB, 참조 → 객체 2번) |
| 크기 | 고정 (`final length`) | 유동 (내부 1.5배 확장) |
| API | 없음 — 검색·삭제·shift 를 반복문으로 직접 | `add`·`remove`·`contains`·Stream |
| 쓰는 곳 | 크기가 정해진 대량 숫자, 성능 핵심부, 라이브러리 내부 | 일반 비즈니스 코드 대부분 |

배열 필드를 그대로 내보내는 게터의 위험은 [[shallow-immutability-defensive-copy]], `Arrays.asList` 의 함정은 [[arrays-aslist-pitfalls]].

> [!WARNING]
> **오답 코너**
> - **"배열은 기본형"** — **참조형(객체)** 이다. 담긴 원소 타입과 배열 자체의 타입은 다르다.
> - **"`int[] b = a; b[0] = 99;` 후 `a[0]` 은 1"** — **99**. 복사된 건 숫자가 아니라 참조다. 원소가 `int` 라도 배열은 객체라 앨리어싱이 일어난다.
> - **"`int[]` 의 칸은 참조를 담는다"** — 참조인 건 **변수**다. `int[]` **칸에는 값 자체**가 있다.
> - **"`x.equals(y)` 는 내용 비교라 true"** — 배열은 `equals` 를 재정의하지 않아 **참조 비교 → false**.
> - **"`List.of` 는 객체에 담지 않아서 값이 그대로 보인다"** — `List.of` 도 객체다. **`toString` 을 재정의**해서 그렇게 보인다.

## 면접 답변

> "배열은 원소가 int 라도 그 자체는 힙에 생기는 객체라서 변수에는 참조가 들어 있습니다. 그래서 배열 변수를 다른 변수에 대입하면 같은 배열을 공유하게 되고, 따로 복사하려면 Arrays.copyOf 나 clone 을 써야 합니다. length 는 final 필드라 크기를 바꿀 수 없고, equals 나 toString 을 재정의하지 않아서 Arrays.equals, Arrays.toString 을 씁니다. 숫자를 대량으로 다루거나 크기가 정해져 있으면 박싱 비용이 없는 배열이, 크기가 변하고 추가·삭제·검색이 필요하면 ArrayList 가 맞습니다."

## 복습 체크

- [ ] 배열이 참조형이라는 증거 두 가지는?
- [ ] 변수·객체·칸 세 층에 각각 무엇이 들어 있는가(`int[]` 와 `String[]` 비교)?
- [ ] `int[] b = a; b[0] = 99;` 후 `a[0]` 과 `a == b` 는? 따로 복사하는 방법은?
- [ ] `length` 에 괄호가 없는 이유와, 그것이 크기 고정과 어떻게 연결되는가?
- [ ] 내용이 같은 두 배열의 `==`, `equals`, `Arrays.equals` 결과는?
- [ ] 배열과 ArrayList 를 고르는 기준 두 가지씩은?

## 관련

[[arraylist-internals]] · [[arrays-aslist-pitfalls]] · [[primitive-vs-reference-type]] · [[java-pass-by-value]] · [[shallow-immutability-defensive-copy]] · [[equals-hashcode-contract]] · [[variable-kinds-initialization]]
