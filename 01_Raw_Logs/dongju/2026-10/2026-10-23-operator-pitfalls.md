---
date: 2026-10-23
type: raw
author: dongju
tags: [연산자함정, operator-pitfalls, 후위증가, post-increment, 전위증가, pre-increment, "i=i++", 증감연산자, 문자열연결, string-concatenation, 연산순서, evaluation-order, 왼쪽결합, left-associative, 삼항연산자, ternary-operator, 조건연산자, conditional-operator, 삼항타입결정, 언박싱NPE, unboxing-npe, 오토박싱, autoboxing, getOrDefault, 정수나눗셈, integer-division, 버림, truncation, 0쪽으로버림, 비율계산, 성공률, 퍼센트, double캐스트, 계산전형변환, 면접답변]
topic: "연산자 함정 네 가지 — i = i++ 는 5, + 는 왼쪽부터 String 을 만나면 연결, 기본형과 래퍼가 섞인 삼항은 언박싱 NPE, 정수 나눗셈은 0 쪽으로 버려 비율이 0.0 이 된다"
summary: "i = i++ 는 i++ 가 5 를 꺼내 두고 i 를 6 으로 올린 뒤 대입이 5 로 덮어써 5 가 된다. + 는 왼쪽부터 계산하고 피연산자에 String 이 나오는 순간부터 연결이라 1 + 2 + \"3\" 은 \"33\", \"1\" + 2 + 3 은 \"123\" 이다. 삼항 연산자의 두 결과가 기본형과 래퍼면 전체 타입이 기본형이 되어 null 쪽이 선택될 때 언박싱 NPE 가 난다. 정수 나눗셈은 반올림이 아니라 0 쪽으로 버리므로 success / total * 100 은 3/4 = 0 → 0.0 이 되고, 계산 전에 피연산자 하나를 double 로 만들어야 한다."
source: session
distilled: true
distilled_at: 2026-10-23
---

## 배운 개념

### 1. 후위 증가 — `i = i++` 는 5

```java
int i = 5;
i = i++;
System.out.println(i);   // 5
```

```
① i++ 가 지금 값 5 를 임시로 꺼내 둔다
② i 를 6 으로 올린다            ← +1 은 여기서 이미 끝남
③ i = (꺼내 둔 5)              ← 대입이 6 을 5 로 덮어씀
```

- 같은 변수를 한 문장에서 증가시키고 대입하면 착시가 생긴다 → `i++` 는 **단독 문장**으로만.

### 2. 문자열 연결 — 왼쪽부터, String 을 만나는 순간부터 연결

```java
System.out.println(1 + 2 + "3");   // "33"   : (1 + 2) = 3 → 3 + "3"
System.out.println("1" + 2 + 3);   // "123"  : "1" + 2 = "12" → "12" + 3
```

### 3. 삼항 연산자의 숨은 언박싱

```java
Map<String, Integer> stock = new HashMap<>();   // "apple" 키 없음
boolean useDefault = false;

Integer count = useDefault ? 0 : stock.get("apple");
// 컴파일러가 바꿔 쓴 모습
Integer count = Integer.valueOf( useDefault ? 0 : stock.get("apple").intValue() );
//                                               ↑ null.intValue() → NPE
```

- 두 결과가 기본형(`0`) + 래퍼(`Integer`) → 삼항 **전체 타입은 기본형 `int`** → null 쪽이 선택되면 언박싱 NPE.
- 좌변이 `Integer` 라 null 을 담을 수 있을 것 같지만, **대입 전에** 터진다.

```java
Integer count = useDefault ? Integer.valueOf(0) : stock.get("apple");   // 둘 다 Integer → 언박싱 없음
int count2   = stock.getOrDefault("apple", 0);                          // null 자체를 없앰
```

- 연결: [[2026-10-22-unboxing-npe-dto-wrapper]]

### 4. 정수 나눗셈 — 0 쪽으로 버린다

```
 3 / 4  =  0.75  →  0
 7 / 2  =  3.5   →  3
-7 / 2  = -3.5   → -3     (내림이면 -4, 0 쪽으로 버려서 -3)
```

```java
int success = 3;
int total   = 4;
double rate = success / total * 100;
// ① 3 / 4       → int 나눗셈 → 0
// ② 0 * 100     → int 곱셈   → 0
// ③ double 대입 → 확대 변환   → 0.0
```

- 에러·경고 없이 성공률 75% 가 **0.0** → 통계 화면 "성공률 0%" 로 늦게 발견되는 버그.

```java
double rate = (double) success / total * 100;   // 75.0 — 캐스트 하나면 충분(다른 쪽도 승격)
double rate = success * 100.0 / total;          // 75.0 — double 리터럴로 시작
double rate = (double) (success / total) * 100; // 0.0 ❌ — 괄호 안 int 나눗셈이 먼저
```

- 마지막 줄은 `(long)(50_000 * 50_000)` 과 같은 함정 — **계산 전에** 형변환 ([[2026-10-22-type-conversion-promotion]]).

### 면접 답변

> "자주 실수하는 지점은 정수 나눗셈과 삼항 연산자입니다. 정수끼리 나누면 소수점이 버려져서 int 성공 건수를 전체 건수로 나누면 double 에 담아도 0 이 되기 때문에, 나누기 전에 한쪽을 double 로 바꿔야 합니다. 또 삼항 연산자에서 한쪽이 기본형이고 다른 쪽이 래퍼면 전체가 기본형으로 맞춰져서, 래퍼 쪽이 null 이면 언박싱 NPE 가 납니다."

## 오답·헷갈린 점

- 👍 `i = i++` → 5, "먼저 값을 돌려주고 다음에 +1" — 결과 정답. 단 +1 이 **대입보다 먼저** 끝나고 대입이 덮어쓴다는 순서는 보충받음.
- 👍 `1 + 2 + "3"` = 33, `"1" + 2 + 3` = 123 — 정답.
- 👍 삼항 전체 타입이 int 로 맞춰진다 — 정답. 단 **그 결과(언박싱 NPE)** 까지 말하지 못함.
- **전**: 삼항 결과가 `Integer` 에 담길 때 "**자동형변환**" → **후**: **오토박싱**. 형변환(확대·축소)은 기본형끼리, 기본형↔래퍼는 박싱/언박싱.
- **전**: `success / total * 100` = **0.75** → **후**: `* 100` 을 빠뜨린 값이라 어떤 경우에도 나올 수 없다(모순). 실제는 **0.0**.
- **전**: `3 / 4` = **1** (반올림으로 이해) → 추적 ① 1, ② 100, ③ 100.0 → **후**: 정수 나눗셈은 **0 쪽으로 버림** → 0, 0, 0.0.
- 👍 고치는 법 "계산 전에 형변환을 미리" — `(double) success / (double) total * 100` 정답(캐스트 하나로도 충분).
- 실무 경험상 연산자 버그는 "없다"고 답함 → 위 시나리오들이 면접·실무 단골이므로 체크리스트로 보관.

## Q&A

**Q. `int i = 5; i = i++;` 후 i 는?**
A. 5. i++ 가 5 를 꺼내 두고 i 를 6 으로 올린 뒤, 대입이 5 로 덮어쓴다.

**Q. `1 + 2 + "3"` 과 `"1" + 2 + 3` 은?**
A. "33" 과 "123". 왼쪽부터 계산, String 을 만나는 순간부터 연결.

**Q. `Integer count = flag ? 0 : map.get(key)` 가 위험한 이유는?**
A. 삼항 전체 타입이 int 가 되어 map.get 이 null 이면 언박싱 NPE. 두 결과를 Integer 로 맞추거나 getOrDefault 를 쓴다.

**Q. `double rate = success / total * 100` 이 0.0 인 이유와 해결은?**
A. int 나눗셈 3/4 = 0(버림). 나누기 전에 `(double) success` 로 만든다.

## 다음 학습

- 이진 탐색 `(low + high) / 2` 오버플로와 `low + (high - low) / 2`, `>>> 1`
- `Math.floorDiv` / `floorMod` 와 음수 나머지(`-7 % 2 == -1`)
- 복합 대입 연산자의 숨은 캐스트 복습 (`byte b += 10`)
