---
date: 2026-10-23
type: raw
author: dongju
tags: [배열, array, 자바배열, java-array, 참조형, reference-type, 배열은객체, array-is-object, int배열, "int[]", null가능, new, 앨리어싱, aliasing, 참조복사, 배열복사, Arrays.copyOf, clone, 배열칸, element, 기본값, default-value, length, final필드, 고정크기, fixed-size, equals미재정의, Object.equals, Arrays.equals, toString, Arrays.toString, "[I@", hashCode, Arrays.hashCode, HashSet배열키, 면접답변]
topic: 배열은 원소가 int 여도 참조형 객체다 — 대입은 참조를 복사해 앨리어싱이 생기고, length 는 final 필드라 크기가 고정이며, equals·toString·hashCode 를 재정의하지 않는다
summary: 배열 변수에는 힙의 배열 객체를 가리키는 참조가 들어 있어 null 이 가능하고 new 로 만든다. 칸의 내용은 원소 타입이 정해 int[] 칸엔 값 자체, String[] 칸엔 참조가 들어 있고, 객체 안의 칸이라 기본값(0/false/null)을 받는다. int[] b = a 는 숫자가 아니라 참조를 복사하므로 b[0] = 99 가 a[0] 도 바꾸며(a == b 는 true), 따로 복사하려면 Arrays.copyOf 나 clone 으로 새 객체를 만든다. length 는 final 필드라 괄호가 없고 크기를 바꿀 수 없다. 배열은 Object 의 equals(참조 비교)·toString([I@해시)·hashCode 를 그대로 쓰므로 내용 비교·출력은 Arrays.equals·Arrays.toString 을 쓴다.
source: session
distilled: false
---

## 배운 개념

### 1. 배열은 참조형 객체 — 세 층으로 본다

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

- 증거: `int[] x = null;` 은 컴파일된다(기본형은 null 불가). 그리고 `new int[3]` 처럼 `new` 로 만든다.
- 칸은 **객체 안의 칸**이라 필드처럼 기본값을 받는다 → `new int[3]` 의 `c[1] == 0` (지역변수처럼 컴파일 에러 아님).
- "위치는 그릇을 따라가고, 내용은 타입이 정한다" 규칙이 배열에도 그대로 적용된다 ([[primitive-vs-reference-type]]).

### 2. 대입은 참조 복사 → 앨리어싱

```java
int[] a = {1, 2, 3};
int[] b = a;
b[0] = 99;
System.out.println("a[0] = " + a[0]);       // a[0] = 99   (실제 실행 결과)
System.out.println("a == b : " + (a == b)); // a == b : true
```

- `int[] b = a` 는 숫자 1, 2, 3 이 아니라 **변수 a 에 들어 있던 참조**를 복사한다 → 둘 다 같은 객체.
- `Member m2 = m1; m2.age = 30;` 과 완전히 같은 구조 ([[java-pass-by-value]]).

따로 복사하려면 **새 배열 객체**를 만든다:

```java
int[] c = Arrays.copyOf(a, a.length);   // 또는 a.clone()
c[0] = -1;
// copy: a[0] = 99, c[0] = -1, a == c : false   (실제 실행 결과)
```

### 3. `length` 는 `final` 필드 → 크기 고정

```java
int[] arr = new int[3];
arr.length = 5;   // ❌ 컴파일 에러: cannot assign a value to final variable length
```

- 메서드가 아니라 필드라서 괄호가 없다. 만들 때 한 번 정해지고 바뀌지 않는다 = 배열 크기 고정의 정체.
- ArrayList 의 `size()` 는 크기가 변하니 매번 물어보는 메서드.

### 4. 배열은 `Object` 의 메서드를 재정의하지 않는다

```java
int[] x = {1, 2, 3};
int[] y = {1, 2, 3};
x == y                 // false — 다른 객체
x.equals(y)            // false — Object.equals 는 this == obj
Arrays.equals(x, y)    // true  — 칸을 하나씩 비교

System.out.println(x);                   // [I@1b6d3586 — 타입 이름 [I + @ + 해시코드 16진수
System.out.println(Arrays.toString(x));  // [1, 2, 3]
System.out.println(List.of(1, 2, 3));    // [1, 2, 3] — List 가 toString 을 재정의
```

| | 배열 | 컬렉션 |
|---|---|---|
| `equals` | 참조 비교 → `Arrays.equals` | 내용 비교 |
| `toString` | `[I@…` → `Arrays.toString` | `[1, 2, 3]` |
| `hashCode` | 객체마다 다름 → `Arrays.hashCode` | 내용 기반 |

- 함정: 배열을 `HashSet`·`HashMap` 키로 쓰면 내용이 같아도 다른 키 ([[equals-hashcode-contract]], [[hashmap-internals]]).

### 면접 답변

> "배열은 원소가 int 라도 그 자체는 힙에 생기는 객체라서 변수에는 참조가 들어 있습니다. 그래서 배열 변수를 다른 변수에 대입하면 같은 배열을 공유하게 되고, 따로 복사하려면 Arrays.copyOf 나 clone 을 써야 합니다. length 는 final 필드라 크기를 바꿀 수 없고, equals 나 toString 을 재정의하지 않아서 내용 비교와 출력은 Arrays.equals, Arrays.toString 을 씁니다."

## 오답·헷갈린 점

- **전**: "배열은 **기본형**이다. 바로 값을 넣으면 변수에 담기기 때문에" → **후**: **참조형(객체)**. 담긴 원소의 타입과 배열 자체의 타입은 다르다. 증거: `int[] x = null` 컴파일, `new` 로 생성.
- **전**: `int[] b = a; b[0] = 99;` 후 `a[0]` = **1** (세 번 반복: "a = 1" → 대답 없음 → "b 는 a 의 값 자체를 복사, a 는 1 유지") → **후**: **99**. 복사된 "값"은 숫자가 아니라 **참조**. 실행 결과 `a == b : true` 로 확인. ⚠️ 어제 `Member m2 = m1` 앨리어싱은 맞혔는데 원소가 `int` 라는 이유로 같은 구조를 못 알아봄.
- **전**: (정정 중) "배열 칸은 참조하는 위치를 나타낸다" → **후**: 반대로 섞임. 참조인 건 **변수 a**, `int[]` **칸에는 값 자체**. 세 층(변수/객체/칸)으로 구분.
- **전**: `c[1]` 기본값 — 답하지 않음 → **후**: 0. 배열 칸은 객체 안의 칸이라 기본값을 받는다.
- **전**: `length` = "배열 객체가 들고 있는 값… 미리 구현해 놓은 것" → **후**: 방향은 맞음. 정확히는 **`final` 필드** → 크기 고정의 정체.
- 👍 `x == y` false — 정답.
- **전**: `x.equals(y)` = true ("실제 값을 비교") → **후**: 배열은 `equals` 미재정의 → `Object.equals` = 참조 비교 → **false**. `Arrays.equals`.
- **전**: 배열 출력 = "참조하는 위치를 출력" → **후**: `Object.toString()` = 타입 이름 + `@` + 해시코드(주소 그 자체는 아님). `Arrays.toString`.
- **전**: `List.of(1,2,3)` 이 `[1, 2, 3]` 으로 찍히는 이유 = "객체에 담지 않았기 때문" → **후**: `List.of` 도 **객체**. List 가 `toString` 을 **재정의**해서.

## Q&A

**Q. 배열은 기본형인가 참조형인가?**
A. 참조형(객체). 변수엔 참조, null 가능, new 로 생성. 원소가 int 여도 마찬가지.

**Q. `int[] b = a; b[0] = 99;` 후 `a[0]` 은?**
A. 99. 참조가 복사돼 같은 객체를 공유한다(a == b 는 true). 따로 쓰려면 Arrays.copyOf / clone.

**Q. `arr.length` 에 괄호가 없는 이유는?**
A. 메서드가 아니라 final 필드. 그래서 크기도 바꿀 수 없다.

**Q. 내용이 같은 두 배열의 `==`, `equals` 결과는?**
A. 둘 다 false. 배열은 equals 를 재정의하지 않는다. 내용 비교는 Arrays.equals.

## 다음 학습

- 배열 공변성: `Object[] arr = new String[3]; arr[0] = 1;` → `ArrayStoreException`, 제네릭 불공변과 비교
- 2차원 배열 = 배열의 배열(행마다 별개 객체, 길이가 다를 수 있음)
