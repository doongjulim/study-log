---
date: 2026-10-07
type: raw
author: dongju
tags: [HashMap, 해시맵, 해시 테이블, hash table, table, 버킷, bucket, hashCode, 해시코드, equals, 해시 충돌, hash collision, 체이닝, separate chaining, Node next, 인덱스 계산, hash & (n-1), 해시 섞기, hash spreading, treeify, 트리화, 레드블랙 트리, red-black tree, TREEIFY_THRESHOLD, resize, 리사이즈, rehash, load factor, 부하율, 0.75, 시간복잡도, 컬렉션, collection, Map]
topic: HashMap 이 키로 O(1) 조회를 하는 원리(hashCode → 인덱스)와 해시 충돌 처리(체이닝·equals·treeify·resize)
summary: HashMap 내부도 Node<K,V>[] 배열이며, 키의 hashCode 를 배열 크기 범위로 줄여(hash & (n-1)) 칸 번호를 계산하므로 넣을 때와 꺼낼 때 같은 칸으로 바로 간다(평균 O(1)). 충돌 시 같은 칸에 next 로 연결(체이닝)하고 hash 비교 + equals 로 키를 골라내며, 최악의 경우 O(n) 은 Java 8 treeify(O(log n))와 0.75 초과 시 2배 resize 로 방어한다.
source: session
distilled: false
---

## 배운 개념

### 1. 내부도 결국 배열

```java
transient Node<K,V>[] table;

static class Node<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;   // 같은 칸의 다음 노드 (단방향)
}
```

배열은 정수 인덱스로만 접근 가능 → **키를 칸 번호로 바꾸는 계산기**가 필요 → `hashCode()`.

### 2. 키 → 칸 번호

```java
"name".hashCode()   // → 3373707 (문자열 내용으로 계산한 int)

// 단순화: 배열 크기 16 일 때
3373707 % 16        // → 11

// 실제 구현
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);  // 상위 비트 섞기
}
index = (n - 1) & hash;   // n 은 2의 거듭제곱 → % 보다 빠른 비트 연산
```

- 같은 내용의 키는 **언제나 같은 hashCode** → put 때 계산한 칸을 get 때도 똑같이 다시 계산 가능.
- (참고: 상위 비트 섞기 때문에 `"name"` 의 실제 인덱스는 11 이 아니라 8. 원리는 "해시값을 배열 크기 범위로 줄인다" 로 동일.)
- 만약 "마지막 칸에 순서대로" 넣으면 get 때 위치를 알 수 없어 처음부터 비교 → O(n) → List 와 다를 바 없음.

### 3. get 흐름

1. `hash(key)` → 해시값
2. `(n-1) & hash` → 칸 번호
3. `table[index]` 로 바로 이동
4. 체인을 따라가며 **hash 비교 → `equals()` 로 키 비교** → 값 반환

```java
// 단순화한 getNode
Node<K,V> e = table[k];
while (e != null) {
    if (e.hash == hash &&          // ① int 비교 (빠른 1차 필터)
        e.key.equals(key)) {       // ② 키 내용 비교 (확정)
        return e.value;
    }
    e = e.next;
}
return null;
```

평균 **O(1)** — n 이 100이든 200이든 단계 수가 같다.

### 4. 해시 충돌과 체이닝

```java
"Aa".hashCode()  // 2112
"BB".hashCode()  // 2112   ← 내용이 다른데 해시값이 같다
```

- int 는 약 43억 가지, 키는 무한 → 충돌은 필연. `& (n-1)` 로 줄이면 칸 충돌은 더 잦다.
- 같은 칸에 `next` 로 줄줄이 연결 = **Separate Chaining**.

```
table[k]  [Aa | hash=2112 | first] → [BB | hash=2112 | second] → null
```

- **equals 를 빼고 hash 만 비교하면** `get("BB")` 가 첫 노드 `Aa` 에서 멈춰 **`"first"` 를 반환하는 버그**.
- → equals 를 재정의하면 hashCode 도 반드시 같이 재정의 (equals true → hashCode 같음). ([[equals-hashcode-contract]])

### 5. 최악의 경우와 방어

```java
@Override
public int hashCode() { return 1; }   // 모든 키가 같은 칸
```

```
table[1]  [k1] → [k2] → … → [k100만]    → get = O(n)
```

| 방어 | 조건 | 효과 |
|---|---|---|
| **Treeify** (Java 8+) | 한 칸 체인 ≥ 8 (`TREEIFY_THRESHOLD`) & 테이블 ≥ 64 | 레드-블랙 트리로 변환 → 최악 **O(log n)**. ≤ 6 이면 다시 리스트. 테이블 < 64 면 트리 대신 resize |
| **Resize** | 원소 수 > 용량 × **0.75** (load factor) | 테이블 **2배** + 전 원소 재배치(rehash) → 충돌 감소. 기본 용량 16 |

→ ArrayList 의 1.5배 확장과 같은 발상(배수 확장 → 분할 상환).

## 오답·헷갈린 점

- **전**: 키를 몇 번 칸에 넣나? → "가장 **마지막 칸**에 넣는다" → **후**: `hashCode()` 로 **칸 번호를 계산**한다. 그래야 꺼낼 때 다시 계산해서 바로 간다.
  - ⚠️ 9/10 에 equals/hashCode 규약을 공부하며 "HashMap 3단계" 를 이미 다뤘는데도 바로 떠올리지 못함 → 회상 약함, 복습 필요.
- **전**: 충돌 시 "노드에 먼저 저장된 값의 해시코드를 저장해서 비교" → **후**: hash 저장·비교는 실제 동작이 맞다(`final int hash`). 하지만 **두 개를 함께 담는 구조**는 `next` 로 잇는 **연결 리스트(체이닝)**.
- **전**: 체인 탐색은 "next 가 있는지 확인, 없으면 호출, 있으면 노드 **값**을 비교" → **후**: next 유무와 무관하게 **현재 노드부터** 확인, 비교 대상은 **값(value)이 아니라 키(key)**.
- **정답 맞힘**: hash 만 비교하면 `Aa` 에서 끝나 잘못된 값 반환 / 전부 한 칸에 몰리면 O(n).

## Q&A

- **Q. 문자열 키로 배열의 몇 번 칸을 정하나?**
  A. `hashCode()` → 상위 비트 섞기 → `(n-1) & hash`.
- **Q. `map.get("name")` 의 단계와 복잡도는?**
  A. 해시 계산 → 칸 이동 → equals 확인 → 반환. 평균 O(1).
- **Q. 해시가 같은 두 키는 어떻게 저장하나?**
  A. 같은 칸에 `next` 로 연결(체이닝).
- **Q. 왜 equals 가 필요한가?**
  A. 해시가 같아도 키가 다를 수 있어서. 없으면 엉뚱한 값 반환.
- **Q. 모든 키가 한 칸이면?**
  A. O(n). Java 8+ 는 8개 이상이면 트리화해 O(log n).

## 다음 학습

- `resize()` 시 Java 8 의 lo/hi 분할 (rehash 없이 원래 자리 or 원래+oldCap)
- 왜 테이블 크기를 2의 거듭제곱으로 유지하나 (`tableSizeFor`)
- 가변 객체를 키로 쓰면 생기는 유령 원소 (→ 기존 노트 [[equals-hashcode-contract]])
- `ConcurrentHashMap` 의 동시성 보장 방식 (CAS + bin 단위 synchronized)
