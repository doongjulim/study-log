---
date: 2026-10-23
type: raw
author: dongju
tags: [배열vsArrayList, array-vs-arraylist, ArrayList, 선택기준, 기본형저장, 박싱비용, boxing-cost, ArrayList<int>불가, 제네릭참조형만, 메모리, memory, 간접참조, 고정크기, 유동크기, 풍부한API, contains, remove, 방어적복사, defensive-copy, private-final배열, 게터노출, clone, List.copyOf, Arrays.asList, 고정크기리스트, fixed-size-list, UnsupportedOperationException, 원본배열감쌈, write-through, "Arrays.asList(int[])", "List<int[]>", size1, 배열자동박싱없음, Arrays.stream, boxed, 면접답변]
topic: 배열은 기본형을 박싱 없이 담아 메모리·속도에 유리하고 ArrayList 는 크기가 유동적이고 API 가 풍부하다 — 그리고 배열 게터 노출과 Arrays.asList 의 두 함정
summary: ArrayList 는 제네릭이라 참조형만 담아 숫자 100만 개면 Integer 객체 100만 개가 생기지만(약 20MB, 간접 접근 2번), int[] 는 값 그대로 약 4MB 다. 반면 배열은 length 가 final 이라 크기가 고정이고 contains·remove 같은 API 가 없어 일반 비즈니스 코드는 ArrayList 가 기본이다. private final int[] 필드를 게터로 그대로 내보내면 바깥에서 내부 상태를 바꿀 수 있어 생성자와 게터에서 clone 하거나 List.copyOf 를 쓴다. Arrays.asList 는 원본 배열을 감싼 고정 크기 리스트라 set 은 원본까지 바꾸고 add 는 UnsupportedOperationException 이며, Arrays.asList(int[]) 는 배열이 자동 박싱되지 않아 List<int[]>(size 1)가 된다.
source: session
distilled: false
---

## 배운 개념

### 1. 선택 기준

| | 배열 | ArrayList |
|---|---|---|
| 기본형 저장 | `int[]` 값 그대로 (100만 개 약 4MB, 읽기 1번) | `ArrayList<int>` 불가 → `Integer` 박싱 (약 20MB, 참조 → 객체 2번) |
| 크기 | 고정 (`length` 는 `final`) | 유동 (내부에서 1.5배 확장: [[arraylist-internals]]) |
| API | 없음 — contains·삭제·shift 를 반복문으로 직접 | `add`·`remove`·`contains`·`indexOf`·Stream·`Collections` |
| 쓰는 곳 | 크기가 정해진 대량 숫자, 성능 핵심부, 라이브러리 내부 | 일반 비즈니스 코드 대부분 |

- 메모리 수치 근거: [[primitive-vs-reference-type]].

### 2. 함정 ① — `private final` 배열을 게터로 그대로 내보내기

```java
public class Order {
    private final int[] quantities;

    public Order(int[] quantities) { this.quantities = quantities; }
    public int[] getQuantities()   { return quantities; }
}

Order order = new Order(new int[]{1, 2, 3});
order.getQuantities()[0] = 999;   // 주문 내부의 첫 수량이 999 로 바뀜
```

- `final` 은 **참조만 고정**한다. 게터가 참조를 그대로 내보내 바깥이 같은 배열 객체를 쥔다(앨리어싱).
- 들어올 때와 나갈 때 **모두** 복사한다 — 생성자도 복사하는 이유: 호출자가 넘긴 배열을 계속 들고 있다 바꿀 수 있다.

```java
public Order(int[] quantities) { this.quantities = quantities.clone(); }   // 들어올 때
public int[] getQuantities()   { return quantities.clone(); }             // 나갈 때
```

- 또는 불변 리스트 `List.copyOf()` 로 보관 ([[shallow-immutability-defensive-copy]]).

### 3. 함정 ② — `Arrays.asList` 는 원본 배열을 감싼 고정 크기 리스트

```java
String[] names = {"a", "b"};
List<String> list = Arrays.asList(names);
list.set(0, "z");     // ✅ 되고, 원본 names[0] 도 "z" 로 바뀜
list.add("c");        // ❌ java.lang.UnsupportedOperationException
```

(실제 실행 결과: `set 후 names[0] = z`, `add → java.lang.UnsupportedOperationException`)

- 감싼 배열의 `length` 가 `final` 이니 크기를 바꿀 수 없다.
- 크기가 바뀌는 리스트가 필요하면: `new ArrayList<>(Arrays.asList(names))`

### 4. 함정 ③ — `Arrays.asList(int[])` 는 `List<int[]>`

```java
int[] nums = {1, 2, 3};
System.out.println(Arrays.asList(nums).size());      // 1
Integer[] boxed = {1, 2, 3};
System.out.println(Arrays.asList(boxed).size());     // 3
```

- `T...` 의 `T` 에 `int` 는 올 수 없다. 오토박싱은 `int` 하나 → `Integer` 하나에만 일어나고 **`int[]` → `Integer[]` 자동 변환은 없다** → `T = int[]`, 배열 통째가 원소 하나.
- 컴파일 에러도 없어 알아차리기 어렵다. 기본형 배열 → 리스트: `Arrays.stream(nums).boxed().toList()`.

### 면접 답변

> "숫자를 대량으로 다루거나 크기가 정해져 있으면 박싱 비용이 없는 기본형 배열이 유리하고, 크기가 변하거나 추가·삭제·검색이 필요한 일반적인 경우에는 ArrayList 를 씁니다. 배열을 필드로 둘 때는 final 이어도 내부가 바뀔 수 있어서 게터와 생성자에서 복사본을 주고받고, Arrays.asList 는 원본 배열을 감싼 고정 크기 리스트라 add 가 안 된다는 점을 주의합니다."

## 오답·헷갈린 점

- 👍 "ArrayList 는 참조형만 가능" — 정답. 단 "100만 개의 참조 위치가 생긴다" → 참조뿐 아니라 참조가 가리킬 **Integer 객체 100만 개**도 생긴다로 보충.
- 👍 배열의 장점 "메모리 절약, 읽기가 단순해 CPU 사용 감소" — 정답.
- 👍 ArrayList 의 장점 "값의 개수가 유동적인 환경에 대처하기 수월" — 정답. 두 번째(풍부한 API)는 보충받음.
- 👍 `private final` 배열 게터 시나리오 → 999, "final 은 참조만 고정" — 정답. (오늘 배열 앨리어싱을 세 번 틀린 뒤 같은 구조를 스스로 적용함) 고치는 방법(생성자·게터 clone)은 보충받음.
- **전**: `Arrays.asList(names).add("c")` = "**바꿀 수 있다**" → **후**: **`UnsupportedOperationException`**. 고정 크기 래퍼. 단 `set` 은 되고 원본까지 바뀐다.
- **전**: `Arrays.asList(int[])` 의 `T` = "**Integer**" → **후**: **`int[]`**. 배열은 자동 박싱되지 않는다 → `List<int[]>`, size 1 (실행으로 확인).

## Q&A

**Q. 배열과 ArrayList 는 언제 구분해 쓰나?**
A. 대량 숫자·고정 크기·성능 핵심부는 배열(박싱 없음), 크기가 변하고 추가·삭제·검색이 필요한 일반 코드는 ArrayList.

**Q. `private final int[]` 를 게터로 그대로 내보내면?**
A. 바깥에서 같은 배열 객체를 쥐어 내부 값을 바꿀 수 있다. 생성자·게터에서 clone 하거나 List.copyOf.

**Q. `Arrays.asList(arr).add(x)` 는?**
A. UnsupportedOperationException. 원본 배열을 감싼 고정 크기 리스트다. set 은 원본까지 바꾼다.

**Q. `Arrays.asList(new int[]{1,2,3}).size()` 는?**
A. 1. T 가 int[] 로 추론돼 List<int[]> 가 된다.

## 다음 학습

- `List.toArray(new String[0])` 관용구와 `toArray()` 가 `Object[]` 를 돌려주는 이유
- `Collections.unmodifiableList` vs `List.copyOf` 복습 ([[shallow-immutability-defensive-copy]])
