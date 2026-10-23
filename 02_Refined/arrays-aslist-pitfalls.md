---
type: refined
slug: arrays-aslist-pitfalls
tags: [Arrays.asList, asList, 고정크기리스트, fixed-size-list, UnsupportedOperationException, 원본배열감쌈, array-backed-list, write-through, set원본반영, add불가, remove불가, "Arrays.asList(int[])", "List<int[]>", size1, 배열자동박싱없음, no-array-autoboxing, 가변인자, varargs, 제네릭참조형만, Arrays.stream, boxed, toList, new-ArrayList, List.of비교, 면접답변]
topic: "Arrays.asList 는 새 리스트가 아니라 원본 배열을 감싼 고정 크기 리스트이고, 기본형 배열을 넘기면 List<int[]> 가 된다"
summary: "Arrays.asList(arr) 는 원본 배열을 List 모양으로 감싼 고정 크기 리스트라 set 은 되고 원본 배열까지 바뀌지만, 감싼 배열의 length 가 final 이라 add·remove 는 UnsupportedOperationException 이다. 크기가 바뀌는 리스트가 필요하면 new ArrayList<>(Arrays.asList(arr)) 로 새로 만든다. 또 T... 의 T 에는 참조형만 올 수 있고 int[] → Integer[] 자동 박싱은 없으므로 Arrays.asList(new int[]{1,2,3}) 은 T = int[] 로 추론돼 List<int[]>, size 1 이 된다. 기본형 배열은 Arrays.stream(nums).boxed().toList() 로 바꾼다."
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Arrays.html#asList(T...)
updated: 2026-10-23
---

# Arrays.asList 의 두 함정

## 개념 정의 — 새 리스트가 아니라 배열을 감싼 것

```java
String[] names = {"a", "b"};
List<String> list = Arrays.asList(names);
list.set(0, "z");     // ✅ 되고, 원본 names[0] 도 "z" 로 바뀜
list.add("c");        // ❌ java.lang.UnsupportedOperationException
```

- `Arrays.asList` 는 새 ArrayList 를 만들지 않는다. 원본 배열을 **List 모양으로 감싼 고정 크기 리스트**다.
- 같은 칸을 쓰므로 `set` 은 **원본까지** 바꾼다(write-through).
- 감싼 배열의 `length` 가 `final` 이라 크기를 바꿀 방법이 없다 → `add`·`remove` 는 예외 ([[java-array]]).
- 크기가 바뀌는 리스트가 필요하면 새로 만든다: `new ArrayList<>(Arrays.asList(names))`

## 작동 원리 — 기본형 배열은 원소 하나가 된다

```java
int[] nums = {1, 2, 3};
System.out.println(Arrays.asList(nums).size());      // 1
Integer[] boxed = {1, 2, 3};
System.out.println(Arrays.asList(boxed).size());     // 3
```

- 시그니처는 `Arrays.asList(T... a)`. `T` 에는 참조형만 올 수 있다.
- 오토박싱은 `int` 하나 → `Integer` 하나에만 일어나고 **`int[]` → `Integer[]` 자동 변환은 없다.**
- 그래서 자바는 `int[]` 배열 **통째를 원소 하나**로 보고 `T = int[]` 로 추론한다 → `List<int[]>`, size 1.
- 컴파일 에러도 경고도 없어서 `contains(1)` 이 항상 false 가 되는 식으로 늦게 드러난다.

```java
List<Integer> list = Arrays.stream(nums).boxed().toList();   // [1, 2, 3]
```

## 트레이드오프 / 한계

| 방법 | 크기 변경 | 원소 변경 | 원본 배열과 연결 |
|---|---|---|---|
| `Arrays.asList(arr)` | ❌ | ⭕ (`set`) | ⭕ 서로 반영 |
| `List.of(...)` | ❌ | ❌ | ❌ (복사) |
| `new ArrayList<>(Arrays.asList(arr))` | ⭕ | ⭕ | ❌ (복사) |

- `Arrays.asList` 는 복사 비용이 없다는 장점이 있지만, 원본과 연결돼 있다는 점을 잊으면 앨리어싱 버그가 된다.

> [!WARNING]
> **오답 코너**
> - **"`Arrays.asList` 결과에 add 할 수 있다"** — **`UnsupportedOperationException`**. 고정 크기 래퍼다. `set` 은 되고 원본까지 바뀐다.
> - **"`Arrays.asList(int[])` 의 T 는 Integer"** — **`int[]`** 다. 배열은 자동 박싱되지 않아 `List<int[]>`, size 1.

## 면접 답변

> "Arrays.asList 는 원본 배열을 감싼 고정 크기 리스트라서 값을 바꾸는 set 은 원본 배열에도 반영되지만 add 나 remove 는 UnsupportedOperationException 이 납니다. 그리고 int 배열을 넘기면 배열 자체가 원소 하나인 List<int[]> 가 되기 때문에, 기본형 배열은 Arrays.stream 으로 박싱해서 바꿉니다."

## 복습 체크

- [ ] `Arrays.asList(arr)` 에서 `set` 과 `add` 의 결과는 각각?
- [ ] 크기가 바뀌는 리스트로 만들려면?
- [ ] `Arrays.asList(new int[]{1,2,3}).size()` 가 1 인 이유는?
- [ ] `Arrays.asList`, `List.of`, `new ArrayList<>(...)` 의 차이를 표로 말할 수 있는가?

## 관련

[[java-array]] · [[arraylist-internals]] · [[shallow-immutability-defensive-copy]] · [[unboxing-npe-wrapper-null]]
