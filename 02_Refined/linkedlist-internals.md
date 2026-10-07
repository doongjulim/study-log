---
type: refined
slug: linkedlist-internals
tags: [LinkedList, 링크드리스트, 연결리스트, linked-list, 이중연결리스트, doubly-linked-list, Node, prev, next, first, last, 순회, traversal, node(int), Deque, 덱, Queue, 큐, ArrayDeque, ListIterator, 캐시지역성, cache-locality, 캐시미스, cache-miss, 메모리오버헤드, memory-overhead, GC부담, 중간삽입, 시간복잡도, 컬렉션, collection, List]
topic: LinkedList 는 이중 연결 Node 의 사슬 — "중간 삽입 O(1)" 은 반만 맞고, 진짜 강점은 맨 앞이며, 메모리·캐시에서 비싸다
summary: LinkedList 는 item/prev/next 를 가진 Node 를 first/last 로 붙잡고 있는 이중 연결 리스트다. 연결 변경 자체는 참조 4개라 O(1) 이지만 인덱스로 위치를 찾는 데 O(n) 이 들어 add(index, x) 전체는 O(n) 이다. 확실한 우위는 맨 앞 삽입·삭제(O(1) vs ArrayList O(n))이고, 원소당 Node 객체 오버헤드와 나쁜 캐시 지역성 때문에 실무에선 ArrayList/ArrayDeque 가 대체로 더 빠르다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/LinkedList.html
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/ArrayDeque.html
updated: 2026-10-07
---

# LinkedList 내부 동작

## 개념 정의

```java
public class LinkedList<E> implements List<E>, Deque<E> {
    Node<E> first;   // 맨 앞 노드
    Node<E> last;    // 맨 끝 노드
    int size;

    private static class Node<E> {
        E item;        // 데이터
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

- **리스트 객체**는 양 끝(`first`/`last`)만, **각 Node** 는 바로 옆 노드만 안다.
- 노드는 힙 **아무 데나 흩어져** 있어도 된다. 배열처럼 연속일 필요가 없다.
- **왜 존재하나**: ArrayList 는 칸이 연속이어야 해서 중간·앞에 끼우려면 뒤를 전부 shift 해야 했다 ([[arraylist-internals]]). 연결로 이으면 이웃 고리만 바꾸면 된다.
- 비유: **기차.** 객차가 땅에 붙어 놓인 게 아니라 연결고리로 이어져 있을 뿐이다.

## 작동 원리

### 1. 연결 변경 — 참조 4개

B 와 C 사이에 X:

```
B.next = X     X.prev = B
C.prev = X     X.next = C

[A] ⇄ [B] ⇄ [X] ⇄ [C] ⇄ [D]
```

원소가 4개든 100만 개든 4개 → **O(1)**. 다른 노드는 움직이지 않는다.

### 2. 인덱스 접근 — 가까운 끝에서 순회

```java
Node<E> node(int index) {
    if (index < (size >> 1)) {        // 앞쪽 절반
        Node<E> x = first;
        for (int i = 0; i < index; i++) x = x.next;
        return x;
    } else {                           // 뒤쪽 절반
        Node<E> x = last;
        for (int i = size - 1; i > index; i--) x = x.prev;
        return x;
    }
}
```

- 값을 비교하는 게 아니라 **몇 칸 갔는지 센다.**
- `prev` 가 있어 가까운 끝에서 출발 → 최악 n/2 → 상수 버리면 **O(n)**.

### 3. "중간 삽입 O(1)" 의 함정

`list.add(500_000, "X")` = **찾기 O(n) + 연결 O(1) = O(n)** — ArrayList 와 같은 등급.

진짜 O(1) 은 **이미 노드를 손에 쥐고 있을 때**뿐이다:

```java
ListIterator<String> it = list.listIterator();
while (it.hasNext()) {
    if (it.next().isBlank()) it.remove();   // 현재 노드를 쥐고 있음 → O(1)
}
```

### 연산별 복잡도

| 연산 | ArrayList | LinkedList |
|---|---|---|
| `get(i)` | **O(1)** | O(n) |
| **맨 앞** 삽입/삭제 | O(n) (shift) | **O(1)** (`first`) |
| 맨 끝 삽입/삭제 | O(1) (add 는 분할상환) | O(1) (`last`) |
| 인덱스로 중간 삽입 | O(n) (shift) | O(n) (찾기) |
| 이터레이터 위치에서 삽입/삭제 | O(n) | **O(1)** |

→ 확실한 우위는 **맨 앞**. 앞에서 빼고 뒤에 넣는 Queue, 양끝을 쓰는 Deque 용도다.

## 트레이드오프 / 한계

### 메모리

```
ArrayList  원소 1개 = 배열 칸 1개 (참조 1개)                      ≈ 4~8 byte
LinkedList 원소 1개 = Node 객체 (객체 헤더 + item + prev + next)  ≈ 24~32 byte
```

원소당 **약 4~6배.** 100만 개면 Node 객체도 100만 개 → **GC 추적 대상**이 그만큼 늘어난다 ([[garbage-collection-reachability]]).

### 캐시 지역성

배열은 연속이라 CPU 가 한 칸을 읽을 때 옆 칸들까지 캐시 라인에 같이 올린다. 노드는 흩어져 있어 `next` 를 따라갈 때마다 **캐시 미스**가 나기 쉽다.

→ **빅오가 같아도 실제 속도는 ArrayList 가 대체로 빠르다.** 큐·덱이 필요해도 배열 기반 원형 버퍼인 **`ArrayDeque`** 가 권장된다.

### 병렬 Stream 소스로는 최악

병렬 Stream 은 소스를 반씩 쪼개 스레드에 나누는데, LinkedList 는 중간 지점을 찾으려면 **앞에서부터 걸어가야(O(n))** 해서 분할 단계에서 다른 스레드가 논다. ArrayList 는 인덱스 계산(O(1))으로 바로 자른다 ([[parallel-stream-pitfalls]]).

> [!WARNING]
> **오답 코너**
> - **"키값을 가진 오브젝트들을 하나의 리스트로 저장"** — List 계열엔 **키가 없다.** 각 원소가 `prev`/`next` 로 이웃을 가리키는 Node 다.
> - **"Node 는 저장된 데이터의 마지막 위치값을 가진다"** — 마지막 위치(`last`)는 **LinkedList 객체**의 필드다. Node 는 `prev`/`next` 만 가진다.
> - **"get(i) 는 리스트 전체를 돌며 next 값을 비교한다"** — 전체가 아니라 **index 까지만**, 비교가 아니라 **칸 수 카운트**다. 가까운 끝에서 출발한다.
> - **"LinkedList 는 중간 삽입·삭제가 O(1) 이라 빠르다"** — **연결만** O(1). 인덱스로 위치를 찾는 O(n) 을 포함하면 전체 O(n) 이다.
> - **"parallel 시 LinkedList 는 next 를 계속 비교해서 느리다"** — 같은 오개념의 재발. 비교가 아니라 **걸으며 카운트**, 손해 보는 단계는 **분할(split)**.
> - **"get(500_000) 은 O(250,000)"** — 빅오에 숫자를 넣지 않는다. n/2 → **O(n)** ([[big-o-notation]]).

## 복습 체크

- [ ] LinkedList 객체와 Node 가 각각 어떤 필드를 가지는가?
- [ ] B 와 C 사이에 X 를 넣을 때 바꾸는 참조 4개를 말할 수 있는가?
- [ ] `get(i)` 가 가까운 끝에서 출발하는 이유와 그래도 O(n) 인 이유는?
- [ ] "중간 삽입 O(1)" 이 반만 맞는 이유와, 진짜 O(1) 이 되는 조건은?
- [ ] LinkedList 가 ArrayList 보다 확실히 유리한 위치는 어디인가?
- [ ] 빅오가 같아도 ArrayList 가 더 빠른 이유 두 가지(메모리/GC, 캐시)는?
- [ ] 큐가 필요할 때 LinkedList 대신 무엇을 쓰는가?

## 관련

[[arraylist-internals]] · [[hashmap-internals]] · [[big-o-notation]] · [[garbage-collection-reachability]] · [[parallel-stream-pitfalls]]
