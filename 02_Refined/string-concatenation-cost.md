---
type: refined
slug: string-concatenation-cost
tags: [문자열연결, string-concatenation, StringBuilder, StringBuffer, 더하기연산, invokedynamic, StringConcatFactory, JEP-280, 바이트코드, bytecode, javap, 시간복잡도, O(n^2), 분할상환, amortized, 버퍼확장, capacity, synchronized, 동기화, 락, 지역변수, check-then-act, 점검후행동, 스레드안전]
topic: + 연산은 컴파일러가 최적화하지만 반복문을 가로지르지 못하며, StringBuffer 는 느려서가 아니라 필요한 구간이 없어서 쓰지 않는다
summary: 하나의 표현식 안의 + 는 invokedynamic 으로 최적화되므로 그냥 써도 되지만, 루프에서 누적하면 매번 전체를 복사해 O(n^2) 가 된다. StringBuilder 는 가변 버퍼에 이어 쓰며 분할상환 O(1) 이고, StringBuffer 의 메서드 단위 동기화는 공유되지 않는 곳에선 낭비이고 공유되는 곳에선 부족하다.
contributors: [dongju]
source_refs:
  - https://openjdk.org/jeps/280
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/StringBuilder.html
updated: 2026-09-10
---

# 문자열 연결의 비용

## 개념 정의

자바에는 연산자 오버로딩이 없다. `+` 는 **바이트코드에 존재하지 않고** 컴파일러가 치환한다.

```java
String one(String name) { return name + "님 환영합니다"; }
```
```
0: aload_1
1: invokedynamic #7,  0     // makeConcatWithConstants:(Ljava/lang/String;)Ljava/lang/String;
6: areturn
```

- Java 8까지: `new StringBuilder().append(...).append(...).toString()`
- Java 9+: `invokedynamic` → `StringConcatFactory` 가 런타임에 더 최적화된 코드 생성 (JEP 280)

**즉 `StringBuilder` 를 직접 써본 적이 없어도 `+` 를 쓸 때마다 그에 준하는 것이 뒤에서 돌고 있다.**

## 작동 원리

### 최적화는 하나의 표현식 안에서만 작동한다

```java
String result = "";
for (int i = 0; i < 10000; i++) { result = result + i; }
```
```
 9: if_icmpge     26
14: invokedynamic #13,  0    // makeConcatWithConstants  ← 루프 안에 있다
23: goto          5
```

`result` 가 이미 5000자일 때 `result + i` 는 5000자짜리 새 배열을 할당하고 **전부 복사한 뒤** 한 글자를 붙인다.
그 5000자짜리는 다음 반복에서 즉시 쓰레기가 된다(GC 압박). `String` 이 불변이라 덧붙일 수가 없기 때문이다 ([[string-immutability]]).

- 10000번 반복 시 복사되는 총 문자 수: `1+2+…+10000 ≈ 5천만` → **O(n²)**
- 실측 (n = 100,000): `+` **5289 ms** vs `StringBuilder` **3 ms** — 약 **1700배**

### 판단 기준은 크기가 아니라 횟수

```java
String big = hugeA + hugeB;                // ✅ 데이터는 커도 연결은 한 번 → + 로 충분
for (...) { result = result + item; }      // ❌ 한 글자씩이어도 만 번 누적 → StringBuilder
```

명시적 `StringBuilder` 가 필요한 실제 자리:

```java
// 동적 쿼리 조립
StringBuilder jpql = new StringBuilder("select n from Notice n where 1=1");
if (title != null)  jpql.append(" and n.title like :title");
if (writer != null) jpql.append(" and n.writer.id = :writerId");

// 목록을 구분자로 잇기
StringBuilder sb = new StringBuilder();
for (String tag : tags) sb.append(tag).append(", ");
```

### StringBuilder 가 빠른 구조적 이유

| | 내부 배열 | 연결 시 |
|---|---|---|
| `String` | `private final byte[] value` — **교체 불가** | 새 배열 할당 + 전체 복사 + 새 객체 생성 |
| `StringBuilder` | `byte[] value` — **가변** | 남은 빈 공간에 **이어 쓰기만** |

버퍼가 차면 "현재 크기 × 2 + 2" 로 확장한다.
10만 자를 담는 동안 `16 → 34 → 70 → 142 → … → 147454` 로 확장은 **13~14번**뿐이고,
복사 총량도 등비급수라 `≈ 2n` 으로 수렴한다.
확장 비용을 전체에 분산시키면 `append()` 한 번당 상수 시간 — **분할상환(amortized) O(1)**, 전체 **O(n)**.
`ArrayList` 가 커질 때 쓰는 것과 같은 전략이다.

## 트레이드오프 / 한계

### StringBuffer 가 죽은 API 인 진짜 이유

```java
public synchronized StringBuffer append(String str) { ... }
```

`synchronized` 가 사는 것은 **스레드 안전성**, 지불하는 것은 **성능**이다.
(불변성이 아니다 — `StringBuilder` 도 `StringBuffer` 도 둘 다 가변이다.)

그런데 실측하면 2000만 회 `append` 기준 **46 ms vs 54 ms, 겨우 17%** 차이다.
현대 JVM 의 JIT 이 경합 없는 락을 대부분 걷어내므로 "수십 배 느리다"는 이제 사실이 아니다.
**"느려서 안 쓴다"는 절반만 맞는 설명이고**, 진짜 이유는 두 갈래다.

**① 공유되지 않는 경우 (대부분)**

```java
public String buildMessage(List<String> items) {
    StringBuffer sb = new StringBuffer();   // 지역변수
    for (String item : items) { sb.append(item).append(", "); }
    return sb.toString();
}
```

10개 스레드가 동시 호출하면 `sb` 는 **10개**가 만들어지고 각각 자기 스택 프레임에서만 참조된다.
메서드 밖으로 새어나가지 않으므로 **공유 자체가 없다** ([[jvm-stack-and-heap]]).
지킬 대상이 없는데 경비원을 세우는 것 — 안전을 사는 게 아니라 비용만 낸다.

**② 진짜로 공유되는 경우 — 그래도 부족하다**

```java
if (sb.length() > 0) {                  // ① 락 획득 → 해제
    sb.deleteCharAt(sb.length() - 1);   // ② 다시 락 획득 → 해제
}
```

**①과 ② 사이에 락이 풀린다.** 다른 스레드가 끼어들면 엉뚱한 위치를 지우거나 예외가 난다.
이런 **점검-후-행동(check-then-act)** 은 개별 메서드 동기화로 보호되지 않고, 결국 호출부에서 더 큰 단위로 동기화해야 한다.

→ **`StringBuffer` 가 정확히 필요한 구간이 거의 존재하지 않는다.**
메서드 단위 동기화는 "안전하다는 착각"만 주고 실제 문제는 못 막는다.
자바는 이 교훈을 반영해 Java 5 에서 `StringBuilder` 를 뒤늦게 추가했다.

> [!WARNING]
> **오답 코너**
> - **"컴파일러가 최적화해주니 `StringBuilder` 는 필요 없다"** — 최적화는 **하나의 표현식 안에서만** 작동한다.
>   루프를 가로지르는 누적에서는 여전히 필수다(실측 1700배).
> - **"기준은 문자열 데이터가 큰 경우"** — **크기가 아니라 반복 횟수(누적 여부)** 다.
> - **"`synchronized` 의 대가는 불변성 상실"** — **성능**이다. 둘 다 애초에 가변이다.
> - **"지역변수 `StringBuffer` 가 무의미한 건 반복문 때문"** — **지역변수라 스레드 간 공유 자체가 안 되기 때문**이다.
> - **"`StringBuffer` 는 `StringBuilder` 보다 수십 배 느리다"** — 현대 JVM 에선 20% 안팎이다.
>   안 쓰는 이유는 성능이 아니라 필요 구간의 부재다.

## 복습 체크

- [ ] `+` 가 바이트코드에서 무엇으로 바뀌는가? Java 8과 9+ 의 차이는?
- [ ] 컴파일러 최적화가 작동하는 범위와 작동하지 않는 범위는?
- [ ] 루프에서 `+` 가 O(n²) 인 이유를 불변성과 연결해 설명할 수 있는가?
- [ ] `StringBuilder` 의 `append` 가 분할상환 O(1) 인 이유는?
- [ ] `StringBuffer` 를 쓰지 않는 두 갈래 이유를 말할 수 있는가?
- [ ] check-then-act 가 왜 메서드 단위 동기화로 보호되지 않는가?

## 관련

[[string-immutability]] · [[jvm-stack-and-heap]] · [[reference-vs-value-equality]]
