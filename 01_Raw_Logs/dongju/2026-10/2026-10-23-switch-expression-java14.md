---
date: 2026-10-23
type: raw
author: dongju
tags: [switch식, switch-expression, 자바14, java14, 문, statement, 식, expression, 문vs식, statement-vs-expression, 값을내놓는다, 삼항연산자, ternary, 화살표case, arrow-case, switch-rule, case규칙, 람다아님, not-lambda, fall-through없음, 다중라벨, case-6-7, 완전성검사, exhaustiveness, does-not-cover-all-possible-input-values, yield, return금지, attempt-to-return-out-of-a-switch-expression, break, enum, default, 새상수추가, 컴파일에러, fail-fast, 콜론형식잔존, 면접답변]
topic: 자바 14 switch 식은 switch 를 값을 내놓는 식으로 만들고, 화살표 형식으로 fall-through 를 없애며, 값을 반드시 내놓아야 하므로 완전성을 컴파일 시점에 검사한다
summary: 식은 실행 결과로 값을 내놓아 변수에 바로 대입할 수 있고 문은 값 없이 동작만 한다. 옛 switch 는 문이라 바깥에 변수를 선언하고 case 마다 대입해야 했지만, 자바 14 switch 식은 String type = switch (day) { case 6, 7 -> "주말"; default -> "평일"; } 처럼 값을 바로 내놓는다. 화살표(->)는 람다가 아니라 fall-through 없는 case 규칙이고, 블록 본문은 yield 로 값을 내놓으며 return 은 메서드를 빠져나가므로 컴파일 에러다. 모든 경우를 다루지 않으면 컴파일 에러(완전성 검사)라서, enum 상수를 전부 나열하면 새 상수 추가 시 default 처럼 조용히 빠지지 않고 컴파일러가 모든 위치를 알려 준다.
source: session
distilled: true
distilled_at: 2026-10-23
---

## 배운 개념

### 1. 문(statement) vs 식(expression)

> **식은 실행 결과로 값을 내놓아 변수에 바로 대입할 수 있고, 문은 값을 내놓지 않고 동작만 한다.**

| | 실행하면 | 변수에 바로 대입 |
|---|---|---|
| `1 + 2` (식) | 값 `3` | `int x = 1 + 2;` ⭕ |
| `if (a > b) { ... }` (문) | 동작만, 값 없음 | `String t = if ...` ❌ 컴파일 안 됨 |
| `a > b ? "큼" : "작음"` (식) | 값 | ⭕ — 삼항은 `if` 의 식 버전 |

옛 switch 는 **문**이라 값을 만들려면:

```java
String type;                 // 바깥에 미리 선언
switch (day) {
    case 6:
    case 7:
        type = "주말";        // case 마다 동작으로서 대입
        break;
    default:
        type = "평일";
}
```

### 2. 자바 14 switch 식

```java
String type = switch (day) {
    case 6, 7 -> "주말";
    default   -> "평일";
};
```

- **식**: 값을 내놓으므로 바로 대입.
- **화살표(`->`) 형식**: fall-through 없음 → `break` 불필요. 한 case 에 여러 값(`case 6, 7`).
- `->` 는 **람다가 아니다.** case 규칙(switch rule) 문법.

| | 람다 `x -> x * 2` | switch 화살표 `case 6 -> "주말"` |
|---|---|---|
| 만드는 것 | 함수형 인터페이스의 **객체** | 아무 객체도 안 만듦 |
| 매개변수 | 있음 | 없음 |
| 변수에 담기 | `Function<Integer,Integer> f = x -> x * 2;` | 불가 |

- **콜론(`case 1:`) 형식은 자바 14 이후에도 남아 있고 여전히 fall-through** 한다. fall-through 가 없는 건 화살표 형식.

### 3. 완전성 검사 — 값을 반드시 내놓아야 한다

```java
String type = switch (day) {    // day 는 int
    case 6, 7 -> "주말";
    case 1, 2, 3, 4, 5 -> "평일";
};
// 컴파일 에러 (실제 javac 결과):
// error: the switch expression does not cover all possible input values
```

- "없으면 null" 규칙은 불가능 — `int code = switch (...)` 라면 null 을 담을 수 없다. 그래서 자바는 모든 경우를 다루지 않으면 **컴파일 자체를 막는다(exhaustiveness)**.

### 4. 블록 본문은 `yield`

```java
String type = switch (day) {
    case 6, 7 -> {
        log.info("주말 요청");
        yield "주말";          // switch 식의 결과값. 메서드는 계속 진행
    }
    default -> "평일";
};
```

```
// return "주말"; 로 쓰면 (실제 javac 결과):
// error: attempt to return out of a switch expression
```

| 키워드 | 빠져나가는 범위 |
|---|---|
| `break` | switch **문**(또는 반복문)만. 값 없음 |
| `yield` | switch **식**에 값을 내놓고 빠져나감 |
| `return` | **메서드 전체** → switch 식 안에서는 금지 |

### 5. 트레이드오프 — `default` vs enum 상수 전부 나열

```java
enum Day { MON, TUE, WED, THU, FRI, SAT, SUN }

// A안
String type = switch (d) {
    case SAT, SUN -> "주말";
    default       -> "평일";
};

// B안 — default 없이 컴파일됨(enum 은 값이 정해져 있음)
String type = switch (d) {
    case SAT, SUN                -> "주말";
    case MON, TUE, WED, THU, FRI -> "평일";
};
```

`Day` 에 `HOLIDAY` 를 추가하면:

| | 결과 | 발견 시점 |
|---|---|---|
| A안 | `HOLIDAY` 가 조용히 "평일" | 운영 중 |
| B안 | `does not cover all possible input values` | **컴파일 때**, `Day` 를 다루는 모든 switch 식에서 |

- 컴파일 에러 = 가장 빠른 fail-fast (배포 전, 위치까지 전부 알려 줌).
- 단서: 이 검사는 switch **식**에서만 강제. switch **문**으로 쓰면 빠진 상수가 있어도 컴파일된다.
- 연결: [[enum-constant-specific-polymorphism]], [[java17-lts-features]]

### 면접 답변

> "자바 14부터 switch 를 값을 내놓는 식으로 쓸 수 있게 됐습니다. 화살표 형식은 fall-through 가 없어서 break 를 쓰지 않아도 되고, 결과를 바로 변수에 대입할 수 있습니다. 또 식은 반드시 값을 내놓아야 하기 때문에 모든 경우를 다루지 않으면 컴파일 에러가 나는데, 그래서 enum 을 다룰 때 default 대신 상수를 전부 나열해 두면 새 상수가 추가됐을 때 컴파일러가 고쳐야 할 곳을 모두 알려 줍니다. 블록 안에서 값을 내놓을 때는 return 이 아니라 yield 를 씁니다."

## 오답·헷갈린 점

- **전**: 자바 14 에서 "switch 문이 **폭포수** 방식이 아니게 되었다" → **후**: 용어는 **fall-through**(폭포수=waterfall 은 개발 방법론과 혼동). 그리고 콜론 형식은 **여전히 fall-through** — 없는 건 **화살표 형식**.
- **전**: "**람다식**을 사용할 수 있게 되었다", "람다식으로 조건에 맞는 경우만 실행" → **후**: `->` 는 람다가 아닌 **case 규칙**. 객체를 만들지 않고 매개변수도 없다.
- 👍 (가) 화살표 형식에서 break 없이도 fall-through 안 함 — 정답.
- **전**: (나) default 없는 int switch 식은 "컴파일된다. 없는 경우 **null 리턴**" → **후**: **컴파일 에러**(완전성 검사). `int` 결과면 null 을 담을 수도 없어 "없으면 null" 규칙 자체가 불가능.
- **전**: 문 vs 식 차이 "모르겠다" → 힌트 후 `String t = if ...` 컴파일 안 됨 ✅, "switch 문은 변수에 담을 수 있게 제공하지 않기 때문" ✅ → 정리: 식은 값을 내놓고, 문은 동작만.
- 👍 enum `HOLIDAY` 추가 시 B안(전부 나열)이 안전 — "조용한 오류보단 에러를 뱉어 개발자가 알게" — 정답. 컴파일 에러라는 점, switch 식에서만 강제된다는 점만 보충.
- **전**: 블록에서 값을 내는 키워드 = "**finish**" → **후**: **`yield`**.
- **전**: "`return` 하면 default 값과 같이 리턴된다" → **후**: **컴파일 에러** `attempt to return out of a switch expression`. return 은 메서드 전체를 빠져나가므로 금지.

## Q&A

**Q. 자바 14 switch 식은 무엇을 바꿨나?**
A. switch 가 값을 내놓는 식이 됐고, 화살표 형식으로 fall-through·break 가 없어졌으며, 모든 경우를 다루도록 컴파일러가 검사한다.

**Q. 문과 식의 차이는?**
A. 식은 값을 내놓아 대입할 수 있고, 문은 동작만 한다. if 는 문, 삼항은 식.

**Q. `case 6, 7 -> "주말"` 의 `->` 는 람다인가?**
A. 아니다. 객체를 만들지 않는 case 규칙 문법.

**Q. int switch 식에서 default 를 빼면?**
A. 컴파일 에러 — the switch expression does not cover all possible input values.

**Q. 블록 본문에서 값을 내놓으려면? return 은?**
A. yield. return 은 메서드를 빠져나가므로 switch 식 안에서 컴파일 에러.

**Q. enum switch 식에서 default 와 전부 나열 중 무엇이 안전한가?**
A. 전부 나열. 새 상수 추가 시 컴파일 에러로 모든 위치를 알려 준다. default 는 조용히 기본 분기로 빠진다.

## 다음 학습

- 자바 21 패턴 매칭 switch(`case Circle c ->`)와 sealed 클래스 완전성 검사
- enum switch 식 vs 상수별 메서드(전략) 선택 기준 ([[enum-constant-specific-polymorphism]] 복습)
