---
date: 2026-09-10
type: raw
author: dongju
tags: [String, 문자열, 불변, immutable, 불변객체, 문자열 상수 풀, String Pool, intern, 상수 폴딩, constant folding, 해시 캐싱, hash cache, 스레드 안전, thread-safe, StringBuilder, StringBuffer, synchronized, 동기화, 락, lock, 문자열 연결, string concatenation, invokedynamic, StringConcatFactory, 바이트코드, bytecode, javap, 시간복잡도, O(n^2), 분할상환, amortized, 버퍼 확장, capacity, check-then-act, 점검 후 행동, 지역변수, 스택, GC]
topic: String 의 불변성이 만들어내는 이득과, 문자열 연결에서 StringBuilder/StringBuffer 를 언제 왜 쓰는가
summary: 불변성 하나가 해시 캐싱·스레드 안전·풀 공유라는 세 이득을 나란히 낳는다. + 연산은 컴파일러가 invokedynamic 으로 최적화하지만 그 최적화는 하나의 표현식 안에서만 작동하므로, 루프를 가로질러 누적하는 자리에서는 O(n^2) 를 O(n) 으로 바꾸기 위해 명시적 StringBuilder 가 필요하다. StringBuffer 는 느려서가 아니라 정확히 필요한 구간이 존재하지 않아서 쓰지 않는다.
source: session
distilled: true
distilled_at: 2026-09-10
---

## 배운 개념

### 1. 불변성 하나가 이득 셋을 낳는다

```
불변(immutable) ─┬─→ 해시 캐싱      한 번 계산한 해시가 영원히 유효
                 ├─→ 스레드 안전     아무도 못 바꾸니 경쟁 상태가 성립 불가
                 └─→ 문자열 풀 공유   같은 리터럴을 여러 곳이 안심하고 공유
```

**원인은 불변성 하나, 결과가 셋이다.** 셋을 서로의 원인으로 엮으면 안 된다
(해시 캐싱 "때문에" 스레드 안전해지는 것이 아니다).

```java
String s = "a";
s = s + "b";
s = s + "c";
```

`"a"` 객체는 끝까지 변하지 않았다. **변한 것은 객체가 아니라 `s` 가 가리키는 대상**이다.
이 3줄로 만들어지는 `String` 객체는 **5개** — 리터럴 `"a"`, `"b"`, `"c"`(상수 풀)와
런타임 연산 결과 `"ab"`, `"abc"`(힙). 저장 영역이 다르다.

### 2. 문자열 상수 풀과 컴파일 타임 상수 폴딩

```java
String a = "abc";
String b = "abc";
String c = "ab" + "c";        // 양쪽 다 리터럴
String e = "ab";
String f = e + "c";           // 변수가 끼면 런타임 연산
String d = new String("abc");
```

실제 실행 결과:

```
a == b : true
a == c : true    <- 컴파일 타임 상수 폴딩. javac 이 "abc" 로 합쳐 풀에 넣는다
a == f : false   <- 런타임 연산 결과는 힙에 새 객체
a == d : false
a == f.intern() : true
```

**어제 배운 `Long` 캐시와 구조가 정확히 같다.** 미리 만들어둔 것을 재사용하면 `==` 가 우연히 `true`,
런타임에 새로 만들면 `false`. 그래서 문자열 비교는 항상 `equals`.

### 3. `private int hash` — 런타임 지연 캐싱

```java
public int hashCode() {
    int h = hash;
    if (h == 0 && !hashIsZero) { h = ...계산...; hash = h; }
    return h;
}
```

컴파일러가 하는 일이 아니라 **런타임에 처음 호출될 때 계산해 필드에 저장**하는 것이다.
긴 문자열을 `HashMap` key 로 100번 조회해도 **실제 계산은 1번**, 나머지 99번은 필드 값을 반환한다.

이 최적화가 가능한 이유가 곧 불변성이다. 어제 본 "가변 필드를 hashCode 에 넣으면 생기는 유령 원소"
(`31 → 32`) 사고가 `String` 에서는 원천적으로 일어날 수 없다. ([[equals-hashcode-contract]] 계열)

### 4. `+` 는 바이트코드에 존재하지 않는다

```java
String one(String name) { return name + "님 환영합니다"; }
```

```
0: aload_1
1: invokedynamic #7,  0     // makeConcatWithConstants:(Ljava/lang/String;)Ljava/lang/String;
6: areturn
```

자바에는 연산자 오버로딩이 없으므로 컴파일러가 `+` 를 치환한다.

- Java 8까지: `new StringBuilder().append(...).append(...).toString()`
- Java 9+: `invokedynamic` → `StringConcatFactory` 가 런타임에 더 최적화된 코드를 생성

**즉 `StringBuilder` 를 직접 써본 적이 없어도, `+` 를 쓸 때마다 그에 준하는 것이 뒤에서 돌고 있었다.**

### 5. 그럼에도 루프에서는 명시적 `StringBuilder` 가 필수

```java
String result = "";
for (int i = 0; i < 10000; i++) { result = result + i; }
```

```
 5: iload_2
 6: sipush        10000
 9: if_icmpge     26
12: aload_1
13: iload_2
14: invokedynamic #13,  0    // makeConcatWithConstants  ← 루프 안에 있다
19: astore_1
20: iinc          2, 1
23: goto          5
```

컴파일러의 최적화는 **하나의 표현식 안에서만** 작동하고 **반복문을 가로지르지 못한다.**

`result` 가 이미 5000자일 때 `result + i` 는 5000자짜리 새 배열을 할당하고 **전부 복사한 뒤** 한 글자를 붙인다.
그리고 그 5000자짜리는 다음 반복에서 즉시 쓰레기가 된다(GC 압박).

- 10000번 반복 시 복사되는 총 문자 수: `1+2+…+10000 ≈ 5천만` → **O(n²)**
- 실측 (n = 100,000):

```
+ 연결       : 5289 ms
StringBuilder:    3 ms      ← 약 1700배
```

**판단 기준은 크기가 아니라 횟수(누적 여부)다.**

```java
String big = hugeA + hugeB;                  // ✅ 데이터는 커도 연결은 한 번 → + 로 충분
for (...) { result = result + item; }        // ❌ 한 글자씩이어도 만 번 누적 → StringBuilder
```

실제로 명시적 `StringBuilder` 가 필요한 자리:

```java
// 동적 쿼리 조립
StringBuilder jpql = new StringBuilder("select n from Notice n where 1=1");
if (title != null)  jpql.append(" and n.title like :title");
if (writer != null) jpql.append(" and n.writer.id = :writerId");

// 목록을 구분자로 잇기
StringBuilder sb = new StringBuilder();
for (String tag : tags) sb.append(tag).append(", ");
```

### 6. `StringBuilder` 가 빠른 구조적 이유

| | 내부 배열 | 연결 시 |
|---|---|---|
| `String` | `private final byte[] value` — **교체 불가** | 새 배열 할당 + 전체 복사 + 새 객체 생성 |
| `StringBuilder` | `byte[] value` — **가변** | 남은 빈 공간에 **이어 쓰기만**. 기존 내용은 건드리지 않음 |

버퍼가 차면 "현재 크기 × 2 + 2" 로 확장한다.
10만 자를 담는 동안 `16 → 34 → 70 → 142 → … → 147454` 로 확장은 **13~14번**뿐이고,
복사 총량도 등비급수라 `≈ 2n` 으로 수렴한다.

확장 비용을 전체 연산에 분산시키면 `append()` 한 번당 상수 시간 —
**분할상환(amortized) O(1)**, 전체 **O(n)**. `ArrayList` 가 커질 때 쓰는 것과 같은 전략이다.

### 7. `StringBuffer` 가 죽은 API 인 진짜 이유

```java
public synchronized StringBuffer append(String str) { ... }
```

`synchronized` 가 사는 것은 **스레드 안전성**, 지불하는 것은 **성능**이다.
(불변성이 아니다 — `StringBuilder` 도 `StringBuffer` 도 둘 다 가변이다.)

그런데 실측하면 2000만 회 `append` 기준:

```
StringBuilder : 46 ms
StringBuffer  : 54 ms      ← 겨우 17% 차이
```

현대 JVM 의 JIT 이 경합 없는 락을 대부분 걷어내므로 "수십 배 느리다" 는 이제 사실이 아니다.
**"느려서 안 쓴다" 는 절반만 맞는 설명이다.** 진짜 이유는 두 갈래다.

**① 공유되지 않는 경우 (99%)**

```java
public String buildMessage(List<String> items) {
    StringBuffer sb = new StringBuffer();   // 지역변수
    for (String item : items) { sb.append(item).append(", "); }
    return sb.toString();
}
```

10개 스레드가 동시 호출하면 `sb` 는 **10개**가 만들어지고, 각각은 자기 스택 프레임에서만 참조된다.
메서드 밖으로 새어나가지 않으므로 **공유 자체가 없다.**
지킬 대상이 없는데 경비원을 세우는 것 — 안전을 사는 게 아니라 비용만 낸다.
([[jvm-stack-and-heap]] 의 "스택은 스레드별, 힙은 공유 1개")

**② 진짜로 공유되는 경우 — 그래도 부족하다**

```java
if (sb.length() > 0) {                  // ① 락 획득 → 해제
    sb.deleteCharAt(sb.length() - 1);   // ② 다시 락 획득 → 해제
}
```

**① 과 ② 사이에 락이 풀린다.** 다른 스레드가 끼어들어 내용을 비우면 엉뚱한 위치를 지우거나 예외가 난다.
이런 **점검-후-행동(check-then-act)** 패턴은 개별 메서드 동기화로 보호되지 않고,
결국 호출부에서 더 큰 단위로 동기화해야 한다.

→ **`StringBuffer` 가 정확히 필요한 구간이 거의 존재하지 않는다.**
메서드 단위 동기화는 "안전하다는 착각" 만 주고 실제 문제는 못 막는다.
자바는 이 교훈을 반영해 Java 5 에서 `StringBuilder` 를 뒤늦게 추가했다.

## 오답·헷갈린 점

| 처음 생각 (오개념) | 정정 |
|---|---|
| `"a"→"ab"→"abc"` 는 객체 3개 | **5개.** 리터럴 `"b"`, `"c"` 도 `String` 객체이고, 리터럴은 풀·연산 결과는 힙으로 **저장 영역도 다르다** |
| `"abc" == "ab" + "c"` 는 `false` | **`true`.** 양쪽이 컴파일 타임 상수라 javac 이 `"abc"` 로 폴딩해 풀에 넣는다 |
| `hash` 캐싱은 컴파일러가 해준다 | **런타임 지연 캐싱.** 첫 호출 때 계산해 필드에 저장하고 이후엔 그 값을 반환 |
| 불변의 두 번째 이득은 "동일성" | **스레드 안전성.** 아무도 못 바꾸니 경쟁 상태가 성립하지 않는다 |
| `synchronized` 의 대가는 불변성 상실 | **성능.** 둘 다 애초에 가변이다. 게다가 현대 JVM 에선 그 성능 차이도 17% 수준 |
| 지역변수 `StringBuffer` 가 무의미한 건 "반복문 때문" | **지역변수라 스레드 간 공유 자체가 되지 않기 때문** |
| **`StringBuilder` 가 필요한 상황은 없다** | **루프·조건문을 가로질러 누적하는 자리에서는 필수.** 컴파일러 최적화는 하나의 표현식 안에서만 작동한다 (실측 1700배) |
| 기준은 "문자열 데이터가 큰 경우" | **기준은 크기가 아니라 반복 횟수(누적 여부)** |
| 불변 → 해시캐싱 → 스레드안전 (직렬 인과) | 불변이 원인이고 **해시캐싱·스레드안전·풀 공유는 나란히 놓인 세 결과** |

## Q&A

**Q. `String` 이 불변이라는 것은 정확히 무슨 뜻인가?**
`s = s + "b"` 를 해도 기존 `"a"` 객체 자체는 변하지 않는다. 변하는 것은 `s` 라는 변수가 가리키는 대상이고, 연결 결과는 매번 새 객체다.

**Q. 왜 불변으로 설계했나?**
해시를 캐싱할 수 있고(HashMap key 로 가장 많이 쓰이는 타입이다), 아무도 상태를 바꿀 수 없으니 동기화 없이 스레드 안전하며, 같은 리터럴을 상수 풀에서 안심하고 공유할 수 있다. 세 이득이 모두 불변성 하나에서 나온다.

**Q. `+` 를 쓰면 안 되나?**
하나의 표현식 안에서라면 써도 된다. 컴파일러가 `invokedynamic`(Java 9+) 또는 `StringBuilder`(Java 8까지)로 바꿔준다. 다만 그 최적화는 반복문을 가로지르지 못하므로, 루프에서 누적하는 자리에서는 명시적 `StringBuilder` 를 써야 한다.

**Q. 루프에서 `+` 가 왜 O(n²) 인가?**
`String` 이 불변이라 덧붙일 수가 없어, 매 반복마다 기존 전체를 새 배열에 복사한 뒤 새 객체를 만든다. n 번 반복하면 복사 총량이 `1+2+…+n ≈ n²/2` 가 된다.

**Q. `StringBuilder` 는 왜 O(n) 인가?**
가변 버퍼를 들고 있어 남은 공간에 이어 쓰기만 한다. 버퍼가 차면 2배로 확장하므로 확장 횟수는 log 규모이고 복사 총량도 `≈ 2n` 으로 수렴한다. `append()` 는 분할상환 O(1).

**Q. `StringBuffer` 는 왜 안 쓰나?**
느려서가 아니다(실측 17% 차이). 지역변수로 쓰면 애초에 공유되지 않아 락이 순수 낭비이고, 진짜로 공유되는 상황이라면 check-then-act 때문에 메서드 단위 동기화만으로는 부족해 어차피 더 큰 단위의 동기화가 필요하다. 정확히 필요한 구간이 존재하지 않는다.

## 다음 학습

- Java 9 **Compact Strings** — 내부 배열이 `char[]` 에서 `byte[]` + `coder` 로 바뀐 이유와 메모리 절감 효과
- `String.intern()` 의 실제 쓰임과 함정 — 힙 문자열을 풀에 넣는 비용, 대량 호출 시의 문제
- text block(Java 15+)과 `String.formatted()` — 가독성 vs 성능
- 불변 객체 설계 일반론 — 값 객체(VO)를 불변으로 만들 때의 이점과 `record` 와의 연결
- study-board 점검: 동적 검색 조건 조립·태그 연결 코드에 루프 `+` 가 남아 있는지 확인
