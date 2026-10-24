---
date: 2026-10-24
type: raw
author: dongju
tags: [String, 문자열, 문자열리터럴, string-literal, new-String, 문자열상수풀, string-constant-pool, string-pool, 힙, heap, 객체공유, 불변, immutable, immutability, final, final클래스, final-vs-불변, 상속금지, toUpperCase, 새객체반환, 상수폴딩, constant-folding, 컴파일타임상수, compile-time-constant, 상수변수, constant-variable, 런타임연결, runtime-concatenation, invokedynamic, makeConcatWithConstants, ldc, javap, 바이트코드, intern, 대표객체, 최대2개, 복습, 면접답변]
topic: 리터럴은 상수 풀에서 공유되고 new 는 항상 힙의 새 객체이며, 공유를 가능하게 하는 건 final 이 아니라 불변성이다 — 풀과 힙을 가르는 기준은 컴파일러가 값을 확신하는가
summary: 같은 리터럴은 상수 풀의 한 객체를 공유하고(a == b true) new String 은 항상 힙에 새 객체를 만든다(최대 2개, 풀에 이미 있으면 1개). 공유가 안전한 건 String 에 내용을 바꾸는 메서드가 없는 불변성 덕분이고, 클래스의 final 은 하위 클래스가 그 불변성을 깨지 못하게 하는 보조 장치다. "java" + "spring" 과 final 상수 변수 연결은 컴파일러가 값을 확신해 컴파일 때 폴딩(ldc "javaspring")되어 풀 객체가 되지만, 일반 변수와의 연결은 실행 중 invokedynamic 으로 만든 힙의 새 객체다. intern() 은 내용이 같은 풀의 대표 객체를 돌려줄 뿐 원래 객체를 옮기지 않는다.
source: session
distilled: true
distilled_at: 2026-10-24
---

## 배운 개념

### 1. 리터럴은 풀에서 공유, `new` 는 항상 새 객체

```java
String a = "java";
String b = "java";
String c = new String("java");

a == b        // true  — 같은 풀 객체 공유
a == c        // false — new 는 힙의 새 객체
a.equals(c)   // true  — 내용 비교
```

```
문자열 상수 풀:   "java"  ◀── a, b
일반 힙:         "java"  ◀── c
```

- `==` 는 변수 안에 담긴 값 자체를 비교한다 — 참조형이면 **참조** ([[reference-vs-value-equality]]).
- `new String("java")` 는 객체를 **최대 2개** 만든다: 괄호 안 리터럴(풀에 없을 때만) + 힙의 새 객체(항상). 위 코드처럼 `a` 가 이미 풀에 만들어 뒀으면 그 줄에서 새로 생기는 건 1개.

### 2. 공유가 안전한 이유 — `final` 이 아니라 불변성

- String 에는 `append`·`setCharAt` 처럼 **자기 내용을 바꾸는 메서드가 없다.** `s.toUpperCase()` 는 `s` 를 바꾸지 않고 **새 String 객체**를 돌려준다.
- 그래서 여러 변수가 한 객체를 공유해도 누군가 내용을 바꿔 다른 쪽이 오염될 일이 없다.
- `final` 과 불변은 다르다:

```java
final StringBuilder sb = new StringBuilder("java");
sb.append("!");   // ✅ 된다 — final 은 참조만 고정
```

- `String` 클래스의 `final`(상속 금지)은 **보조 장치**: 상속해서 내용을 바꾸는 메서드를 추가한 하위 클래스가 생기면 풀에서 공유하던 문자열이 모두 변경될 수 있으므로 그 길을 막는다.
- 연결: [[string-immutability]], [[shallow-immutability-defensive-copy]]

### 3. 풀이냐 힙이냐 — 컴파일러가 값을 확신하는가

```java
String s1 = "javaspring";
String s2 = "java" + "spring";      // s1 == s2  true
String part = "spring";
String s3 = "java" + part;          // s1 == s3  false
final String fpart = "spring";
String s4 = "java" + fpart;         // s1 == s4  true
String s5 = s3.intern();            // s1 == s5  true,  s3 == s5  false
```

실행 결과: `true false true true / s3 == s5: false`

`javap -c` 로 본 바이트코드:

```
 0: ldc   "javaspring"      ← s1
 3: ldc   "javaspring"      ← s2 = "java" + "spring"   (덧셈이 사라짐 = 폴딩)
 6: ldc   "spring"          ← part
10: invokedynamic makeConcatWithConstants   ← s3 = "java" + part   (실행 중 연결)
17: ldc   "javaspring"      ← s4 = "java" + fpart      (덧셈이 사라짐 = 폴딩)
23: invokevirtual String.intern             ← s5
```

| 코드 | 컴파일러가 확신? | 결과 |
|---|---|---|
| `"java" + "spring"` | ⭕ 리터럴끼리 | 컴파일 때 **상수 폴딩** → 풀 |
| `"java" + part` | ❌ 일반 변수(실행 중 바뀔 수 있음) | 실행 중 연결 → **힙의 새 객체** |
| `"java" + fpart` | ⭕ `final` + 리터럴 초기화 = **상수 변수** | 폴딩 → 풀 |

- 어제의 `byte c = 10 + 20`(상수 표현식은 컴파일러가 값을 앎)과 같은 원리 ([[numeric-promotion]]).

### 4. `intern()` — 풀의 대표 객체를 돌려준다

```
s3.intern()
 → 풀에서 "javaspring" 과 내용이 같은 객체를 찾는다
 → 이미 있음 (s1 이 가리키는 객체) → 그 객체를 반환
```

- 힙의 `s3` 를 풀로 옮기는 게 아니다 → `s3 == s5` 는 false. 풀에 없으면 등록한 뒤 그 객체를 반환.

### 면접 답변

> "리터럴로 만든 문자열은 문자열 상수 풀에서 같은 내용이면 같은 객체를 공유하고, new 로 만들면 항상 힙에 새 객체가 생깁니다. 이렇게 공유할 수 있는 건 String 이 불변이라 누가 내용을 바꿀 수 없기 때문입니다. 또 리터럴끼리나 final 상수끼리의 연결은 컴파일러가 미리 계산해서 풀의 객체가 되지만, 변수와 연결하면 실행 중에 새 객체가 만들어집니다."

## 오답·헷갈린 점

- 👍 `a == b` true, `a == c` false, `a.equals(c)` true — 정답(복습). 단 각 객체의 위치(풀/힙)는 답하지 않아 보충.
- **전**: "`==` 는 객체의 **참조 위치**를 비교" → **후**: 변수 안에 담긴 **값**을 비교(참조형이면 참조). 위키 `reference-vs-value-equality` 오답과 같은 계열.
- **전**: `new String("java")` 는 **1개** → **후**: **최대 2개**(풀 리터럴 + 힙). 이 코드에선 풀에 이미 있어 1개도 맞음 — 조건을 붙여 답할 것.
- **전**: 공유하려면 String 은 "**final 성질**"을 가져야 한다 → **후**: **불변성**. `final StringBuilder` 도 append 된다(final ≠ 불변). 클래스 final 은 보조 장치 — 이후 "내용 바꾸는 메서드 없음, toUpperCase 는 새 객체", "상속 허용 시 풀 문자열이 모두 변경될 수 있다" 둘 다 스스로 답함 👍.
- 👍 (1) true (2) false (3) true — 결과 정답.
- **전**: (4) `s3.intern()` 후 `s1 == s5` = **false** → **후**: **true**. intern 은 풀의 대표 객체를 돌려준다. 단 `s3 == s5` 는 false.
- **전**: (1)의 이유 = "**변수에 담기는 시점에** 상수 풀에 올라가기 때문" → **후**: **컴파일할 때** 폴딩(`ldc "javaspring"`). 그 규칙이면 (2) 도 true 여야 한다는 모순.
- 👍 (2)의 이유 = "part 의 값이 같은 값이라는 확신을 할 수 없기 때문" — 방향 정답("자바" → "컴파일러가 컴파일할 때").
- **전**: (3)의 이유 = "컴파일러는 **메소드가 호출하여 생성하는 시점에** 확신한다" → **후**: 컴파일러는 실행하지 않는다. `final` + 리터럴 초기화 = **상수 변수**라 코드만 읽고 확신 → 폴딩.
- ⚠️ 반복 테마: 결과는 맞히는데 "**언제**"(컴파일 시점 vs 실행 시점)를 섞는다 — 클래스 로딩(초기화 시점), 언박싱 NPE(컴파일 에러 vs 런타임), 오버플로(잘림 시점)와 같은 계열. 바이트코드로 직접 확인한 것이 효과적이었음.

## Q&A

**Q. 리터럴과 new 로 만든 String 의 차이는?**
A. 리터럴은 상수 풀에서 같은 내용이면 한 객체를 공유하고, new 는 항상 힙에 새 객체를 만든다.

**Q. `new String("java")` 는 객체를 몇 개 만드나?**
A. 최대 2개(풀 리터럴 + 힙 객체). 풀에 이미 있으면 1개.

**Q. 리터럴 공유가 안전한 이유는?**
A. String 이 불변이라. 클래스 final 은 상속으로 불변성이 깨지는 걸 막는 보조 장치.

**Q. `"java" + "spring"` 과 `"java" + part` 의 차이는?**
A. 앞은 컴파일러가 값을 확신해 컴파일 때 폴딩 → 풀. 뒤는 변수라 실행 중 연결 → 힙의 새 객체. final + 리터럴 초기화면 상수 변수라 다시 폴딩.

**Q. `intern()` 은?**
A. 내용이 같은 풀의 대표 객체를 돌려준다(없으면 등록). 원래 객체를 옮기진 않는다.

## 다음 학습

- `intern()` 을 직접 쓰면 안 되는 이유(풀 크기·성능)와 `-XX:StringTableSize`
- 문자열 연결의 `invokedynamic` / `StringConcatFactory` ([[string-concatenation-cost]] 복습)
