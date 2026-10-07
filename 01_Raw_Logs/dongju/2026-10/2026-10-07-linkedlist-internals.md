---
date: 2026-10-07
type: raw
author: dongju
tags: [LinkedList, 링크드리스트, 연결 리스트, linked list, 이중 연결 리스트, doubly linked list, Node, prev, next, first, last, 순회, traversal, node(int index), Deque, 덱, Queue, 큐, ArrayDeque, ListIterator, 캐시 지역성, cache locality, 캐시 미스, cache miss, 메모리 오버헤드, memory overhead, GC 부담, 시간복잡도, 컬렉션, collection, List]
topic: LinkedList 의 내부 구조(이중 연결 Node)와, "중간 삽입 O(1)" 이 반만 맞는 이유, 메모리·캐시 측면 트레이드오프
summary: LinkedList 는 item/prev/next 를 가진 Node 들을 연결한 이중 연결 리스트로, 연결 변경 자체는 O(1) 이지만 인덱스로 위치를 찾는 데 O(n) 이 들어 add(index, x) 전체는 O(n) 이다. 진짜 강점은 first/last 로 바로 접근하는 맨 앞 삽입·삭제(O(1))이고, 원소당 Node 객체 오버헤드와 나쁜 캐시 지역성 때문에 실무에선 ArrayList/ArrayDeque 가 대체로 더 빠르다.
source: session
distilled: false
---

## 배운 개념

### 1. 구조: 이중 연결 리스트

```java
public class LinkedList<E> {
    Node<E> first;   // 맨 앞 노드
    Node<E> last;    // 맨 끝 노드
    int size;

    private static class Node<E> {
        E item;        // 실제 데이터
        Node<E> prev;  // 바로 앞 노드
        Node<E> next;  // 바로 뒤 노드
    }
}
```

```
first                                       last
  ↓                                           ↓
[A] ⇄ [B] ⇄ [C] ⇄ [D]
 prev=null                           next=null
```

- **LinkedList 객체**가 `first`/`last` 를 들고 있고, **각 Node** 는 바로 옆 노드만 안다.
- 노드는 힙 메모리 **아무 데나 흩어져** 있어도 된다 (배열처럼 연속일 필요 없음).
- 비유: **기차.** 객차가 땅에 붙어 일렬로 놓인 게 아니라 연결고리로 이어져 있을 뿐. 사이에 끼우려면 고리만 다시 건다.

### 2. 중간 삽입 — 연결은 참조 4개

B 와 C 사이에 X 삽입:

```
B.next = X     X.prev = B
C.prev = X     X.next = C

[A] ⇄ [B] ⇄ [X] ⇄ [C] ⇄ [D]
```

원소가 4개든 100만 개든 참조 4개 → **연결 자체는 O(1)**. ArrayList 의 shift 가 없다.

### 3. 인덱스 접근 — 순회 O(n)

```java
Node<E> node(int index) {
    if (index < (size >> 1)) {        // 앞쪽 절반이면
        Node<E> x = first;
        for (int i = 0; i < index; i++) x = x.next;
        return x;
    } else {                           // 뒤쪽 절반이면
        Node<E> x = last;
        for (int i = size - 1; i > index; i--) x = x.prev;
        return x;
    }
}
```

- `first` 에서 next 를 **index 번 따라가며 칸 수를 센다** (값 비교 아님).
- 이중 연결이라 **가까운 쪽 끝**에서 출발 → 최악 n/2 번 → 상수 버리면 **O(n)**.

### 4. 면접 함정: "LinkedList 중간 삽입은 O(1)"

`linkedList.add(500_000, "X")` =
1. 찾기: 50만 번째 노드까지 순회 → **O(n)**
2. 연결: 참조 4개 → **O(1)**

전체 **O(n) + O(1) = O(n)** → ArrayList 와 같은 등급.
진짜 O(1) 은 **이미 노드를 손에 쥐고 있을 때**뿐 — 예: `ListIterator` 로 순회하며 `it.add()` / `it.remove()`.

### 5. 진짜 강점: 맨 앞

| 위치 | ArrayList | LinkedList |
|---|---|---|
| **맨 앞** 삽입/삭제 | **O(n)** (shift) | **O(1)** (`first`) |
| 맨 끝 삽입/삭제 | O(1) (add 는 분할 상환) | O(1) (`last`) |
| `get(i)` | O(1) | O(n) |

→ 앞에서 빼고 뒤에 넣는 **Queue**, 양끝을 쓰는 **Deque** 용도. `LinkedList implements List, Deque`.

### 6. 빅오 밖의 트레이드오프: 메모리 & 캐시

```
ArrayList  원소 1개 = 배열 칸 1개 (참조 1개)                     ≈ 4~8 byte
LinkedList 원소 1개 = Node 객체 (객체 헤더 + item + prev + next)  ≈ 24~32 byte
```

- 원소당 **약 4~6배** 메모리.
- **GC 부담**: 원소 100만 개 = 객체 100만 개 추가 → GC 추적 대상 증가.
- **캐시 지역성**: 배열은 연속이라 CPU 가 한 번 읽을 때 옆 칸까지 캐시 라인에 같이 올림. 노드는 흩어져 있어 next 를 따라갈 때마다 **캐시 미스**.
- 그래서 빅오가 같아도 실제 속도는 ArrayList / **`ArrayDeque`**(배열 기반 원형 덱)가 대체로 빠르다 → 큐·덱이 필요해도 `ArrayDeque` 권장.

## 오답·헷갈린 점

- **전**: 키값을 가진 모든 오브젝트를 하나의 리스트로 저장 → **후**: List 계열에는 **키가 없다**. 각 원소가 `prev`/`next` 로 이웃을 가리키는 Node.
- **전**: Node 의 두 필드는 "저장된 데이터의 마지막 위치값" → **후**: "마지막 위치" 는 **LinkedList 객체의 `last`** 필드. Node 는 `prev`(앞) / `next`(뒤) 만 가진다.
- **전**: get(i) 는 리스트 전체를 호출해서 next 값을 **비교** → **후**: 전체가 아니라 index 까지만, 비교가 아니라 **몇 칸 갔는지 카운트**. 가까운 끝에서 출발.
- **전**: get 복잡도를 `25만번` / `O(250,000)` 으로 답함 → **후**: 숫자 대입 금지, n/2 → 상수 버림 → **O(n)**. (→ [[big-o-notation-basics]] 참고)
- **전**: "중간 삽입 O(1)" 을 그대로 믿을 뻔 → **후**: 찾기 O(n) 포함 시 전체 O(n).

## Q&A

- **Q. LinkedList Node 는 무엇을 들고 있나?**
  A. `item`, `prev`, `next`. 리스트 객체 자신은 `first`, `last`, `size`.
- **Q. B 와 C 사이에 X 를 끼우려면?**
  A. `B.next=X`, `X.prev=B`, `X.next=C`, `C.prev=X` — 4개 (정답 맞힘).
- **Q. `get(500_000)` 은 어떻게 찾나?**
  A. 가까운 끝(first/last)에서 next/prev 를 따라 칸 수를 세며 이동. O(n).
- **Q. LinkedList 가 ArrayList 보다 확실히 유리한 경우?**
  A. **맨 앞** 삽입·삭제 (O(1) vs O(n)). 맨 끝은 둘 다 O(1).
- **Q. 메모리 관점 차이?**
  A. 원소마다 Node 객체(헤더+참조3개)를 더 만든다 → 4~6배 메모리, GC 부담, 캐시 미스.

## 다음 학습

- `ArrayDeque` 내부 구조 (원형 버퍼, head/tail 인덱스) — 왜 LinkedList 보다 빠른가
- `ListIterator` 로 순회 중 삽입/삭제, `ConcurrentModificationException` 과 `modCount`
- CPU 캐시 라인(64B)과 배열 순회 성능 실측 (JMH)
