---
date: 2026-10-07
type: raw
author: dongju
tags: [ArrayList, 어레이리스트, 동적 배열, dynamic array, elementData, Object 배열, 랜덤 접근, random access, grow, 용량 확장, capacity, 1.5배, Arrays.copyOf, System.arraycopy, shift, 원소 이동, 분할 상환, amortized, 시간복잡도, time complexity, Vector, Hashtable, 컬렉션, collection, List]
topic: ArrayList 의 내부 구조(Object[])와 용량 확장·삽입/삭제 비용, 그리고 add 가 분할 상환 O(1) 인 이유
summary: ArrayList 는 Object[] elementData 를 감싼 동적 배열이라 get(i) 는 주소 계산 한 번으로 O(1)이다. 꽉 차면 1.5배 새 배열을 만들어 복사하는데, 배수로 키우기 때문에 add 는 분할 상환 O(1)이고 +1씩 키우면 전체 O(n²)로 폭발한다. 맨 앞 삽입/삭제는 같은 배열 안에서 나머지를 shift 하므로 O(n)이다.
source: session
distilled: false
---

## 배운 개념

### 1. 내부는 그냥 배열이다

```java
transient Object[] elementData;   // ArrayList 내부
private int size;
```

- `get(i)` → `elementData[i]` 반환이 전부.
- 배열은 메모리에 칸이 **연속**으로 붙어 있어서 `시작주소 + 인덱스 × 칸크기` 계산 한 번으로 위치가 나온다 → **O(1)** (랜덤 접근).
- List 에는 **키가 없다.** `0, 1, 2…` 는 내가 정한 키가 아니라 `add` 순서대로 자동으로 붙는 **위치(인덱스)**.

### 2. 왜 ArrayList 가 필요한가 — 배열은 크기 고정

```java
String[] arr = new String[3];
arr[3] = "D";   // ArrayIndexOutOfBoundsException
```

배열은 생성 순간 크기가 고정된다. 바로 뒤 메모리는 다른 객체가 쓰고 있을 수 있어 **제자리에서 늘릴 수 없다.**

### 3. 꽉 찼을 때: 새 배열 + 복사 (grow)

```java
// ArrayList.grow() (Java 8+)
int newCapacity = oldCapacity + (oldCapacity >> 1);  // 1.5배
elementData = Arrays.copyOf(elementData, newCapacity);
```

```
기존: [ A | B | C ]          ← 참조 끊김 → GC 대상
          ↓ 복사 (Arrays.copyOf)
새것: [ A | B | C | D |   |   ]
```

- 기본 용량 10. `new ArrayList<>()` 시점엔 빈 배열이고 **첫 add 때** 10으로 할당(lazy).
- 증가율 비교:

| 컬렉션 | 증가 방식 |
|---|---|
| ArrayList | **1.5배** |
| Vector | 2배 |
| Hashtable | 2배 + 1 |

### 4. 분할 상환(amortized) O(1)

복사는 O(n) 이지만 **얼마나 크게 늘리느냐**가 전체 비용을 결정한다. n = 10,000 번 add 기준:

| 전략 | 총 복사량 | 복잡도 |
|---|---|---|
| +1 칸씩 | 1+2+…+9,999 ≈ **5,000만** | 전체 O(n²) |
| ×1.5 | 10+15+22+…+9,369 ≈ **2.8만** (확장 18회) | add 1회당 O(1) |

- 1.5배 시퀀스: `10 → 15 → 22 → 33 → 49 → 73 → 109 → 163 → 244 → 366 → 549 → 823 → 1234 → 1851 → 2776 → 4164 → 6246 → 9369 → 14053`
- 2.8만 / 1만 회 = add 1회당 평균 약 2~3 회 복사. **n 이 1억이 돼도 이 비율은 그대로** → 상수 → O(1).
- 비유: **휴대폰 할부.** 큰 복사 한 번(일시불)을 그 뒤 따라오는 복사 없는 add 들(배수로 늘린 만큼 많음)에 나눠 얹으면 1회당 상수. +1씩 늘리면 공짜 add 가 딱 1번뿐이라 나눠 낼 데가 없다.

### 5. 맨 앞 / 맨 끝 삽입·삭제

```
add(0, "X") 전:  [ A | B | C | D |   ]
                   ↘   ↘   ↘   ↘        ← 같은 배열 안에서 한 칸씩 뒤로 (System.arraycopy)
add(0, "X") 후:  [ X | A | B | C | D ]
```

- 빈칸이 있으면 **새 배열을 만들지 않는다.** 같은 배열 안에서 shift. 새 배열은 꽉 찼을 때만.
- 맨 끝 삭제는 밀 원소 **0개**:

```java
elementData[--size] = null;   // GC 위해 null 처리
```

### 6. 연산별 복잡도

| 연산 | 복잡도 | 이유 |
|---|---|---|
| `get(i)` | O(1) | 주소 계산 한 번 |
| `add(x)` (끝) | 분할 상환 O(1) | 가끔 1.5배 확장 + 복사 |
| `add(0,x)` / `remove(0)` | **O(n)** | 뒤 원소 전부 shift |
| `remove(size-1)` | O(1) | 밀 원소 없음 |

## 오답·헷갈린 점

- **전**: ArrayList 는 내부적으로 키값을 둬서 저장한다 → **후**: 키는 Map 의 개념. List 는 **인덱스(위치)**만 있다.
- **전**: 내부를 **스택** 구조로 들고 있다 → **후**: 평범한 `Object[]` 배열. 스택은 LIFO 라 맨 위만 접근 가능 → 인덱스 랜덤 접근과 정반대 성질.
- **전**: "데이터 크기에 따라 길이를 조정한다" (추상적) → **후**: 배열은 제자리 확장 불가. **더 큰 새 배열 생성 → 기존 원소 복사 → elementData 교체 → 기존 배열 GC**.
- **전**: 새 오브젝트를 만들고 두 오브젝트를 **병합** → **후**: 병합이 아니라 **기존 → 새 배열로 복사**, 기존은 버려짐.
- **전**: 새 배열은 현재의 **2배+1** → **후**: ArrayList 는 **1.5배**. 2배+1 은 `Hashtable` 의 rehash 공식.
- **전**: 1.5배면 확장 8번 정도 → **후**: 10→1만 기준 **18번**. 그리고 중요한 건 횟수가 아니라 **총 복사량(≈2.8만)**.
- **전**: 분할 상환 1회당 2~3 을 **O(n²)** 라고 부른다 → **후**: n 과 무관하게 일정 → **분할 상환 O(1)**. O(n²) 는 +1 전략의 전체 비용.
- **전**: 맨 앞 삽입 시 새 오브젝트를 만들고 나머지를 복사 → **후**: 빈칸 있으면 **같은 배열 안에서 shift**. "복사"보다 "이동(shift)".
- **전**: 맨 끝 삭제는 1개 이동 → **후**: **0개**. 마지막 칸 null + size--.

## Q&A

- **Q. ArrayList 는 내부적으로 데이터를 어디에 저장하나?**
  A. `Object[] elementData` 배열. `get(i)` 는 `elementData[i]` 이고 주소 계산 한 번이라 O(1).
- **Q. 배열을 그냥 쓰면 안 되나?**
  A. 배열은 크기 고정이라 넘치면 `ArrayIndexOutOfBoundsException`. ArrayList 는 꽉 차면 1.5배 새 배열로 옮겨 탄다.
- **Q. 1칸씩 늘리면 왜 안 되나?**
  A. 매 add 마다 전체 복사 → 총 n²/2 → O(n²). 배수로 늘리면 총 복사량이 O(n) 이라 add 1회당 분할 상환 O(1).
- **Q. 100만 개 ArrayList 의 `add(0, x)` 비용은?**
  A. 나머지 원소를 전부 한 칸씩 shift → O(n). 맨 끝 삭제는 O(1).

## 다음 학습

- `ensureCapacity()` / `new ArrayList<>(initialCapacity)` 로 확장 비용 미리 없애기 — 언제 쓰나
- `trimToSize()` 와 메모리 회수
- `ArraysSupport.newLength()` (Java 17) 에서 최대 용량·오버플로 처리
