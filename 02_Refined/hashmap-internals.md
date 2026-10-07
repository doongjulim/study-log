---
type: refined
slug: hashmap-internals
tags: [HashMap, 해시맵, 해시테이블, hash-table, table, 버킷, bucket, hashCode, 해시코드, equals, 해시충돌, hash-collision, 체이닝, separate-chaining, Node, next, 인덱스계산, 해시섞기, hash-spreading, 비트연산, treeify, 트리화, 레드블랙트리, red-black-tree, TREEIFY_THRESHOLD, resize, 리사이즈, rehash, load-factor, 부하율, 0.75, 2의거듭제곱, 시간복잡도, 컬렉션, collection, Map]
topic: HashMap 은 키의 hashCode 로 배열 칸 번호를 계산해 평균 O(1) 로 찾고, 충돌은 체이닝 + equals 로 가르며, 최악은 treeify 와 resize 로 방어한다
summary: HashMap 내부도 Node<K,V>[] 배열이다. 키의 hashCode 를 섞은 뒤 (n-1) & hash 로 칸 번호를 계산하므로 put 과 get 이 같은 칸으로 바로 간다(평균 O(1)). 충돌하면 같은 칸에 next 로 연결하고 hash 비교 후 equals 로 키를 확정하며, 원소 수가 용량×0.75 를 넘으면 2배 resize, 한 칸 체인이 8 이상이면 레드-블랙 트리로 바꿔 최악을 완화한다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/HashMap.html
  - https://openjdk.org/jeps/180
updated: 2026-10-07
---

# HashMap 내부 동작

## 개념 정의

```java
transient Node<K,V>[] table;   // 내부도 결국 배열

static class Node<K,V> {
    final int hash;     // 계산해 둔 해시 (재계산·빠른 비교용)
    final K key;
    V value;
    Node<K,V> next;     // 같은 칸의 다음 노드 (단방향)
}
```

**왜 필요한가** — List 에서 특정 값을 찾으려면 처음부터 비교해야 한다(O(n)). 키만 보고 **칸 번호를 계산**할 수 있으면 배열의 O(1) 랜덤 접근([[arraylist-internals]])을 그대로 쓸 수 있다. 그 계산기가 `hashCode()` 다.

> 만약 "들어온 순서대로 마지막 칸에" 넣는다면, 꺼낼 때 위치를 알 방법이 없어 처음부터 비교해야 한다 → O(n) → List 와 다를 게 없다.

## 작동 원리

### 1. 키 → 칸 번호

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);   // 상위 16비트를 하위로 섞기
}
int index = (n - 1) & hash;   // n = table.length (항상 2의 거듭제곱) → % n 과 같지만 더 빠름
```

```java
"name".hashCode()   // 3373707
// 단순화하면 3373707 % 16 = 11
// 실제로는 상위 비트 섞기 후 (16-1) & hash = 8
```

- 같은 내용의 키는 **언제나 같은 hashCode** → put 때 계산한 칸을 get 때도 그대로 다시 계산한다.
- 상위 비트를 섞는 이유: `(n-1) &` 은 **하위 비트만** 보므로, 하위 비트가 비슷한 해시들이 한 칸에 몰리는 걸 줄인다.

### 2. get — 칸 이동 → 체인 순회 → equals 확정

```java
// 단순화한 getNode
Node<K,V> e = table[(n - 1) & hash];
while (e != null) {
    if (e.hash == hash &&                                  // ① int 비교 — 빠른 1차 필터
        (e.key == key || key.equals(e.key))) {             // ② 키 내용 확정
        return e.value;
    }
    e = e.next;
}
return null;
```

비교 대상은 **값(value)이 아니라 키(key)** 이고, next 유무와 무관하게 **현재 노드부터** 확인한다. 평균 **O(1)**.

### 3. 해시 충돌 — 체이닝

```java
"Aa".hashCode()   // 2112
"BB".hashCode()   // 2112   ← 내용이 다른데 같은 해시
```

int 는 약 43억 가지, 키는 무한 → 충돌은 필연이다 ([[equals-hashcode-contract]] 의 비둘기집 원리). `(n-1) &` 로 줄이면 **칸 충돌**은 훨씬 잦다.

```
table[k]  [Aa | hash=2112 | "first"] → [BB | hash=2112 | "second"] → null
```

같은 칸에 `next` 로 연결 = **Separate Chaining**.

**② equals 를 빼면?** `get("BB")` 가 첫 노드 `Aa` 에서 hash 2112 일치로 멈춰 **`"first"` 를 반환하는 버그.** 해시가 같다고 같은 키가 아니다 → equals 재정의 시 hashCode 도 함께 ([[equals-hashcode-contract]]).

### 4. resize — 0.75 를 넘으면 2배

| 항목 | 값 |
|---|---|
| 기본 용량 | 16 |
| load factor | 0.75 |
| 확장 조건 | `size > capacity × 0.75` (16 이면 13번째 put) |
| 확장 | **2배** + 모든 노드 재배치 |

- 칸이 넉넉해지면 체인이 짧아져 충돌이 준다. ArrayList 1.5배 확장과 같은 **배수 확장 → 분할상환** 발상 ([[big-o-notation]]).
- 용량이 2배가 되면 인덱스 비트가 하나 늘 뿐이라, 각 노드는 `(hash & oldCap) == 0` 이면 **원래 자리**, 아니면 **원래 자리 + oldCap** 으로 간다. 저장된 `hash` 를 쓰므로 `hashCode()` 를 다시 부르지 않는다.
- 크기를 미리 알면 초기 용량을 줘서 resize 를 피한다 (`expected / 0.75 + 1`).

### 5. 최악의 경우 — treeify

```java
@Override public int hashCode() { return 1; }   // 모든 키가 한 칸
```

```
table[1]  [k1] → [k2] → … → [k100만]   → get = O(n)
```

| 조건 | 동작 |
|---|---|
| 한 칸 체인 ≥ 8 (`TREEIFY_THRESHOLD`) & 테이블 ≥ 64 | 체인 → **레드-블랙 트리** |
| 한 칸 체인 ≥ 8 이지만 테이블 < 64 | 트리 대신 **resize** 먼저 |
| 트리 노드 ≤ 6 (`UNTREEIFY_THRESHOLD`) | 다시 리스트로 |

트리는 **hash 값 → (같으면) `Comparable` 순서**로 정렬한다. 그래서
- 해시가 서로 다른데 칸만 겹친 경우, 또는 키가 `Comparable`(String 등)인 경우 → 최악 **O(log n)**.
- **해시가 전부 같고 키가 `Comparable` 도 아니면** 정렬 기준이 없어 양쪽 서브트리를 다 뒤져야 해서 여전히 **O(n)** 에 가깝다.

(Java 8 에서 도입 — 공격자가 같은 해시의 문자열 키를 대량으로 보내는 hash flooding DoS 방어 목적, JEP 180.)

### 연산별 복잡도

| 상황 | get / put |
|---|---|
| 평균 (해시 분포 양호) | **O(1)** |
| 한 칸에 몰림, 리스트 체인 | O(n) |
| 한 칸에 몰림, 트리화 + 정렬 가능 | O(log n) |

## 트레이드오프 / 한계

- **평균 O(1) 은 hashCode 품질에 달려 있다.** 분포가 나쁘면 느려질 뿐 틀리지는 않지만, equals/hashCode 규약을 어기면 **데이터를 잃는다** ([[equals-hashcode-contract]]).
- **순서 보장 없음.** 순회 순서는 칸 번호 순이고 resize 마다 바뀔 수 있다 → 삽입 순서가 필요하면 `LinkedHashMap`, 정렬이 필요하면 `TreeMap`.
- **가변 키 금지.** 넣은 뒤 해시에 쓰인 필드가 바뀌면 다른 칸을 계산해 영영 못 찾는다 (유령 원소).
- **스레드 안전하지 않다.** 동시 put 은 데이터 유실 → `ConcurrentHashMap` ([[sse-server-sent-events]]).
- 빈 칸과 Node 객체 오버헤드로 메모리를 넉넉히 쓴다 (load factor 0.75 = 25% 이상 빈칸 유지).

> [!WARNING]
> **오답 코너**
> - **"키를 배열의 가장 마지막 칸에 넣는다"** — `hashCode()` 로 **칸 번호를 계산**한다. 그래야 꺼낼 때 같은 계산으로 바로 간다.
> - **"충돌하면 먼저 저장된 값의 해시코드를 저장해 비교한다"** — hash 를 저장·비교하는 건 맞지만(1차 필터), **두 개를 함께 담는 구조는 `next` 로 잇는 연결 리스트(체이닝)** 다.
> - **"체인에서 next 가 없으면 호출, 있으면 노드 값을 비교"** — next 유무와 무관하게 **현재 노드부터** 확인하고, 비교 대상은 **값이 아니라 키**다.
> - **"hash 만 비교하면 충분하다"** — `"Aa"`/`"BB"` 처럼 hash 가 같은 다른 키에서 **엉뚱한 값을 반환**한다. 마지막은 반드시 `equals()`.
> - **"Java 8 treeify 덕에 최악도 항상 O(log n)"** — 해시가 전부 같고 키가 `Comparable` 이 아니면 트리에서도 정렬 기준이 없어 **O(n) 에 가깝다.** 상수 `hashCode` 는 트리로도 구제되지 않을 수 있다.
> - **"resize 때 모든 키의 hashCode 를 다시 계산한다"** — 저장된 `hash` 로 원래 자리 / +oldCap 만 가른다.

## 복습 체크

- [ ] 문자열 키가 몇 번 칸에 들어갈지 정해지는 과정(hashCode → 섞기 → `(n-1) &`)을 말할 수 있는가?
- [ ] 테이블 크기를 2의 거듭제곱으로 유지하는 이유는?
- [ ] `map.get(key)` 의 단계와 각 단계에서 hash / equals 의 역할은?
- [ ] 해시가 같은 두 키는 어떻게 저장되고, equals 를 빼면 무슨 버그가 나는가?
- [ ] load factor 0.75, 2배 resize 의 의미와 resize 를 피하는 법은?
- [ ] treeify 조건 3가지(8, 64, 6)와, 트리화해도 O(log n) 이 안 되는 경우는?
- [ ] HashMap 대신 LinkedHashMap / TreeMap / ConcurrentHashMap 을 쓸 상황은?

## 관련

[[equals-hashcode-contract]] · [[arraylist-internals]] · [[linkedlist-internals]] · [[big-o-notation]] · [[string-immutability]] · [[sse-server-sent-events]]
