---
date: 2026-10-24
type: raw
author: dongju
tags: [가변인자, varargs, variable-arguments, "int...", "T...", 배열로묶기, 배열매개변수, array-parameter, "int[]", 참조형배열, 빈배열, empty-array, new-int-0, length0, null전달, NullPointerException, NPE, 맨마지막, 하나만, last-parameter, Arrays.asList, 배열생성비용, 오버로드, SLF4J, 면접답변]
topic: 가변 인자는 인자들을 배열로 묶어 넘기는 문법이다 — 메서드 안에서는 int[] 이고, 인자가 없으면 null 이 아닌 빈 배열이 넘어가며, 맨 뒤에 하나만 쓸 수 있다
summary: int... nums 는 메서드 안에서 int[] (참조형 배열)이라 nums.length·nums[0] 을 쓸 수 있고 배열을 그대로 넘길 수도 있다. 컴파일러는 sum(1, 2, 3) 을 sum(new int[]{1, 2, 3}) 으로, sum() 을 sum(new int[0]) 으로 바꿔 쓰므로 인자가 없어도 nums 는 null 이 아닌 길이 0 배열이라 for 문이 NPE 없이 돈다. 호출자가 일부러 (int[]) null 을 넘길 때만 null 이 되어 NPE 가 난다. 가변 인자는 매개변수 맨 마지막에 하나만 쓸 수 있는데, 그래야 앞의 고정 매개변수부터 채우고 나머지 전부를 배열로 묶는다는 경계가 정해지기 때문이다.
source: session
distilled: false
---

## 배운 개념

### 1. 가변 인자의 정체 — 배열

```java
static int sum(int... nums) {      // 메서드 안에서 nums 는 int[]
    int total = 0;
    for (int n : nums) total += n;
    return total;
}
```

- 증거: 메서드 안에서 `nums.length`, `nums[0]` 을 쓴다. `sum(new int[]{4, 5})` 처럼 배열을 그대로 넘겨도 컴파일된다.
- 배열은 **참조형 객체**다 ([[java-array]]).

컴파일러가 바꿔 쓰는 모습:

| 호출 | 컴파일러가 바꾼 모습 | `nums` | 결과 |
|---|---|---|---|
| `sum(1, 2, 3)` | `sum(new int[]{1, 2, 3})` — 호출마다 새 배열 | 길이 3 배열 | 6 |
| `sum()` | `sum(new int[0])` | **빈 배열**(null 아님) | 0 |
| `sum(new int[]{4, 5})` | 그대로 | 길이 2 배열 | 9 |
| `sum((int[]) null)` | 그대로 | **null** | NPE |

실행 결과:

```
sum() 호출
  nums == null ? false, length = 0
  결과 = 0
sum((int[]) null) 호출
  nums == null ? true
  → NullPointerException
```

- "인자 없음"을 null 이 아니라 **빈 배열**로 표현해 주므로 메서드 작성자가 null 체크 없이 안전하다. 단 호출자가 일부러 null 을 넘기면 진짜 null 이 된다.
- `Arrays.asList(int[])` 가 `List<int[]>` 가 되는 이유도 `T...` 가 사실 `T[]` 이기 때문 ([[arrays-aslist-pitfalls]]).

### 2. 규칙 — 매개변수 맨 마지막에, 하나만

```java
void log(String... messages, int level)   // ❌
```

- `log("a", "b", 3)` 이면 어디까지가 `messages` 이고 어디부터가 `level` 인지 경계를 정할 수 없다. `level` 도 String 이면 완전히 모호.
- 맨 뒤에 있으면 규칙이 단순: **앞의 고정 매개변수부터 채우고 남은 인자는 전부 가변 인자 배열.** 두 개면 첫 번째가 어디서 끝나는지 알 수 없어 하나만 허용.

### 3. (보충) 트레이드오프 — 호출마다 배열 생성

- 아주 자주 불리는 메서드는 고정 개수 오버로드를 함께 둔다. 예: SLF4J 의 `log.debug(String, Object)`, `log.debug(String, Object, Object)` 오버로드는 배열 생성을 피하려는 것.

### 면접 답변

> "가변 인자는 넘긴 인자들을 컴파일러가 배열로 묶어 주는 문법이라 메서드 안에서는 배열로 다룹니다. 인자를 하나도 안 넘기면 null 이 아니라 길이 0인 배열이 넘어가고, 경계를 정할 수 있도록 매개변수의 맨 마지막에 하나만 둘 수 있습니다."

## 오답·헷갈린 점

- **전**: `int... nums` 의 실제 타입 = "**기본 타입**" → **후**: **`int[]`**, 참조형 배열. ⚠️ 어제 "배열은 기본형" 을 정정했는데([[java-array]]) 원소 타입이 `int` 로 보이자 같은 오개념이 **재발**. "원소 타입 ≠ 배열 자체 타입".
- **전**: 인자 없이 `sum()` 을 부르면 `nums` 는 "다른 무언가" → (보기 제시 후) "**null 을 넘긴다**", 그러면서 "length 는 0, NPE" → **후**: null 이면 `nums.length` 에서 NPE 라 0 이 나올 수 없다는 모순. 정답은 **`new int[0]` 빈 배열** → length 0, for 문은 안 돌고 끝 → 결과 0. 실행으로 확인.
- 👍 가변 인자가 맨 뒤·하나만인 이유 = "여러 개 들어왔을 때 어디까지가 String 이고 어디까지가 int 인지 알 수 없기 때문" — 정답.

## Q&A

**Q. 가변 인자의 실제 타입은?**
A. 배열(`int... nums` → `int[]`). 컴파일러가 인자들을 배열로 묶어 넘긴다.

**Q. 인자 없이 호출하면 `nums` 는?**
A. 길이 0인 빈 배열(null 아님). 호출자가 `(int[]) null` 을 넘길 때만 null.

**Q. 가변 인자를 맨 뒤에 하나만 쓸 수 있는 이유는?**
A. 그래야 고정 매개변수를 채운 뒤 나머지 전부를 배열로 묶는다는 경계가 정해진다.

## 다음 학습

- 오버로딩 해석에서 가변 인자가 가장 마지막 순위인 이유
- `@SafeVarargs` 와 제네릭 가변 인자의 힙 오염 경고
