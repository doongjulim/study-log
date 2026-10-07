---
type: refined
slug: arraylist-internals
tags: [ArrayList, 어레이리스트, 동적배열, dynamic-array, elementData, Object배열, 랜덤접근, random-access, 인덱스, index, grow, 용량확장, capacity, 1.5배, Arrays.copyOf, System.arraycopy, shift, 원소이동, 분할상환, amortized, 시간복잡도, Vector, Hashtable, 컬렉션, collection, List, ensureCapacity, 초기용량]
topic: ArrayList 는 Object[] 를 감싼 동적 배열 — 1.5배 확장으로 add 는 분할상환 O(1), 맨 앞 삽입·삭제는 shift 때문에 O(n)
summary: ArrayList 는 Object[] elementData 를 들고 있어 get(i) 가 주소 계산 한 번(O(1))이다. 배열은 제자리 확장이 불가능해 꽉 차면 1.5배 새 배열을 만들어 복사하는데, 배수로 키우기 때문에 add 는 분할상환 O(1) 이고 +1 씩 키우면 전체 O(n²) 로 폭발한다. 빈칸이 있을 때의 맨 앞 삽입·삭제는 새 배열 없이 같은 배열 안에서 나머지를 shift 하므로 O(n) 이다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/ArrayList.html
updated: 2026-10-07
---

# ArrayList 내부 동작

## 개념 정의

```java
transient Object[] elementData;   // 실제 저장소
private int size;                 // 채워진 원소 수 (≠ 배열 길이 = capacity)
```

ArrayList 는 **평범한 배열**을 감싼 동적 배열이다. 키도 스택도 아니다.

- `get(i)` → `elementData[i]` 가 전부.
- 배열은 칸이 메모리에 **연속**으로 붙어 있어 `시작주소 + 인덱스 × 칸크기` 한 번이면 위치가 나온다 → **O(1)** (랜덤 접근).
- List 의 `0, 1, 2…` 는 내가 정한 **키가 아니라** add 순서로 붙는 **위치(인덱스)** 다. 키 → 값은 Map 의 개념 ([[hashmap-internals]]).

**왜 필요한가** — 배열은 생성 순간 크기가 고정된다.

```java
String[] arr = new String[3];
arr[3] = "D";   // ArrayIndexOutOfBoundsException
```

바로 뒤 메모리는 다른 객체가 쓰고 있을 수 있어 **제자리에서 늘릴 수 없다.** ArrayList 는 이 문제를 "더 큰 배열로 이사" 로 푼다.

## 작동 원리

### 1. 확장 (grow) — 새 배열 + 복사

```java
int newCapacity = oldCapacity + (oldCapacity >> 1);   // 1.5배
elementData = Arrays.copyOf(elementData, newCapacity);
```

```
기존: [ A | B | C ]          ← 참조 끊김 → GC 대상
          ↓ 복사
새것: [ A | B | C | D |   |   ]
```

1. 더 큰 새 배열 생성 → 2. 기존 원소 복사 → 3. `elementData` 교체 → 4. 기존 배열은 GC 대상.

- `new ArrayList<>()` 는 빈 배열로 시작하고 **첫 add 때** 기본 용량 10 을 할당한다 (lazy).

| 컬렉션 | 증가 방식 |
|---|---|
| ArrayList | **1.5배** |
| Vector | 2배 |
| Hashtable | 2배 + 1 |
| StringBuilder | 2배 + 2 ([[string-concatenation-cost]]) |

### 2. 왜 배수인가 — 분할상환 O(1)

복사 한 번은 O(n) 이다. 결정적인 건 **얼마나 크게 늘리느냐**다. add 10,000 회 기준:

| 전략 | 총 복사량 | 결과 |
|---|---|---|
| +1 칸씩 | 1 + 2 + … + 9,999 ≈ **5,000만** | 전체 **O(n²)** |
| ×1.5 | 10 + 15 + 22 + … + 9,369 ≈ **2.8만** (확장 18회) | add 1회당 **분할상환 O(1)** |

```
10 → 15 → 22 → 33 → 49 → 73 → 109 → 163 → 244 → 366
→ 549 → 823 → 1234 → 1851 → 2776 → 4164 → 6246 → 9369 → 14053
```

2.8만 / 1만 회 ≈ add 1회당 복사 2~3회. **n 이 1억이 돼도 이 비율은 변하지 않는다** → 상수 → O(1).

> 비유: **휴대폰 할부.** 큰 복사(일시불) 한 번 뒤에는, 배수로 늘린 만큼 **복사 없는 add 가 많이** 따라온다. 그 add 들에 비용을 나눠 얹으면 1회당 상수.
> +1 씩 늘리면 큰 복사 뒤 공짜 add 가 딱 1번이라 나눠 낼 데가 없다.

빅오 읽는 법 자체는 [[big-o-notation]].

### 3. 맨 앞 / 맨 끝 삽입·삭제

```
add(0, "X") 전:  [ A | B | C | D |   ]
                   ↘   ↘   ↘   ↘         ← 같은 배열 안에서 한 칸씩 뒤로 (System.arraycopy)
add(0, "X") 후:  [ X | A | B | C | D ]
```

- 빈칸이 있으면 **새 배열을 만들지 않는다.** 같은 배열 안에서 **shift**. 새 배열은 꽉 찼을 때만.
- 맨 끝 삭제는 밀 원소가 **0개**:

```java
elementData[--size] = null;   // 마지막 칸 비우고 size 감소 (GC 위해 null)
```

### 연산별 복잡도

| 연산 | 복잡도 | 이유 |
|---|---|---|
| `get(i)` / `set(i, x)` | O(1) | 주소 계산 한 번 |
| `add(x)` (끝) | 분할상환 O(1) | 가끔 1.5배 확장 + 복사 |
| `add(0, x)` / `remove(0)` | **O(n)** | 뒤 원소 전부 shift |
| `remove(size - 1)` | O(1) | 밀 원소 없음 |
| `contains(x)` | O(n) | 처음부터 equals 비교 |

## 트레이드오프 / 한계

- **맨 앞 삽입·삭제가 O(n).** 앞에서 빼는 큐 용도엔 부적합 → `ArrayDeque` ([[linkedlist-internals]]).
- **확장 순간의 지연 스파이크.** 평균은 O(1) 이지만 확장이 터지는 그 add 는 O(n). 크기를 알면 `new ArrayList<>(expectedSize)` 나 `ensureCapacity()` 로 확장 자체를 없앤다.
- **남는 용량은 메모리 낭비.** 최대 1/3 가까이 빈칸일 수 있다 (`trimToSize()` 로 회수).
- 그래도 연속 메모리라 **캐시 지역성**이 좋아 실무 기본 List 는 ArrayList 다.

> [!WARNING]
> **오답 코너**
> - **"ArrayList 는 내부적으로 키값을 둬서 저장한다"** — 키는 Map 의 개념이다. List 는 **인덱스(위치)** 뿐이다.
> - **"내부는 스택 구조"** — 평범한 `Object[]` 배열이다. 스택(LIFO)은 맨 위만 접근할 수 있어 인덱스 랜덤 접근과 정반대 성질이다.
> - **"데이터 크기에 따라 길이를 조정한다" / "두 오브젝트를 병합한다"** — 배열은 제자리 확장이 불가능하다. **새 배열을 만들어 복사하고 기존 배열은 버린다.** 병합이 아니라 이사다.
> - **"ArrayList 는 2배+1 로 늘린다"** — **1.5배**(`old + (old >> 1)`). 2배+1 은 `Hashtable` 이다.
> - **"분할상환 add 1회당 2~3 은 O(n²)"** — n 과 무관하게 일정하므로 **분할상환 O(1)**. O(n²) 는 +1 전략의 **전체** 비용이다.
> - **"맨 앞 삽입 시 새 배열을 만들어 복사한다"** — 빈칸이 있으면 **같은 배열 안에서 shift** 한다. 새 배열 생성(확장)과 원소 이동(shift)은 다른 사건이다.
> - **"맨 끝 삭제는 1개 이동"** — **0개**. null 대입 + size 감소뿐이다.

## 복습 체크

- [ ] ArrayList 의 실제 저장소 필드와 `get(i)` 가 O(1) 인 이유를 주소 계산으로 설명할 수 있는가?
- [ ] 배열을 제자리에서 늘릴 수 없는 이유는?
- [ ] 꽉 찼을 때 일어나는 4단계(생성·복사·교체·GC)를 순서대로 말할 수 있는가?
- [ ] ArrayList / Vector / Hashtable 의 증가율을 구분할 수 있는가?
- [ ] +1 씩 vs 1.5배씩 늘릴 때 총 복사량의 차이와, 그게 왜 add 1회당 O(1) 이 되는지 할부 비유로 설명할 수 있는가?
- [ ] `add(0, x)` 와 `remove(size-1)` 의 복잡도와, 각각 몇 개가 이동하는가?
- [ ] 확장(새 배열)과 shift(같은 배열 내 이동)의 차이는?

## 관련

[[linkedlist-internals]] · [[hashmap-internals]] · [[big-o-notation]] · [[string-concatenation-cost]] · [[garbage-collection-reachability]]
