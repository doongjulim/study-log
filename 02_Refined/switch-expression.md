---
type: refined
slug: switch-expression
tags: [switch식, switch-expression, 자바14, java14, 문, statement, 식, expression, 문vs식, statement-vs-expression, 값을내놓는다, 삼항연산자, ternary, 화살표case, arrow-case, switch-rule, case규칙, 람다아님, not-lambda, fall-through없음, 다중라벨, 완전성검사, exhaustiveness, does-not-cover-all-possible-input-values, yield, return금지, attempt-to-return-out-of-a-switch-expression, break, 콜론형식잔존, 면접답변]
topic: "자바 14 switch 식은 switch 를 값을 내놓는 식으로 만들고, 화살표 형식으로 fall-through 를 없애며, 값을 반드시 내놓아야 하므로 완전성을 컴파일 시점에 검사한다"
summary: "식은 실행 결과로 값을 내놓아 변수에 바로 대입할 수 있고 문은 값 없이 동작만 한다. 옛 switch 는 문이라 바깥에 변수를 선언하고 case 마다 대입해야 했지만, switch 식은 값을 바로 내놓는다. 화살표(->)는 람다가 아니라 fall-through 없는 case 규칙이고, 콜론 형식은 여전히 fall-through 한다. 블록 본문은 yield 로 값을 내놓으며 return 은 메서드를 빠져나가므로 컴파일 에러다. 모든 경우를 다루지 않으면 컴파일 에러(완전성 검사)이고, int 결과에 null 을 줄 수 없으니 '없으면 null' 규칙은 불가능하다."
contributors: [dongju]
source_refs:
  - https://openjdk.org/jeps/361
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-15.html#jls-15.28
updated: 2026-10-23
---

# switch 식 (Java 14)

## 개념 정의 — 문(statement) vs 식(expression)

> **식은 실행 결과로 값을 내놓아 변수에 바로 대입할 수 있고, 문은 값을 내놓지 않고 동작만 한다.**

| | 실행하면 | 변수에 바로 대입 |
|---|---|---|
| `1 + 2` (식) | 값 `3` | `int x = 1 + 2;` ⭕ |
| `if (a > b) { ... }` (문) | 동작만, 값 없음 | `String t = if ...` ❌ |
| `a > b ? "큼" : "작음"` (식) | 값 | ⭕ — 삼항은 `if` 의 식 버전 |

옛 switch 는 **문**이라 값을 만들려면 바깥에 변수를 선언하고 case 마다 동작으로서 대입해야 했다(대입을 빠뜨린 case 가 있으면 변수가 빈다).

```java
String type;
switch (day) {
    case 6:
    case 7:
        type = "주말";
        break;
    default:
        type = "평일";
}
```

switch 식은 switch 의 식 버전이다.

```java
String type = switch (day) {
    case 6, 7 -> "주말";
    default   -> "평일";
};
```

## 작동 원리

### 화살표 형식 — fall-through 없음, 람다 아님

- `->` 형식은 fall-through 가 없어 `break` 가 필요 없다. 한 case 에 여러 값(`case 6, 7`)을 쓴다.
- **콜론(`case 1:`) 형식은 자바 14 이후에도 남아 있고 여전히 fall-through 한다** ([[switch-fall-through]]). fall-through 가 없는 것은 화살표 형식이다.
- `->` 는 **람다가 아니다.** 기호만 빌린 case 규칙(switch rule) 문법이다.

| | 람다 `x -> x * 2` | switch 화살표 `case 6 -> "주말"` |
|---|---|---|
| 만드는 것 | 함수형 인터페이스의 **객체** | 아무 객체도 안 만듦 |
| 매개변수 | 있음 | 없음 |
| 변수에 담기 | `Function<Integer,Integer> f = x -> x * 2;` | 불가 |

### 완전성 검사 — 값을 반드시 내놓아야 한다

```java
String type = switch (day) {    // day 는 int
    case 6, 7 -> "주말";
    case 1, 2, 3, 4, 5 -> "평일";
};
// error: the switch expression does not cover all possible input values
```

"없으면 null" 규칙은 불가능하다. `int code = switch (...)` 라면 null 을 담을 수 없기 때문이다. 그래서 모든 경우를 다루지 않으면 컴파일 자체를 막는다. enum 은 상수를 전부 나열하면 `default` 없이 컴파일되고, 새 상수가 추가되면 이 에러로 모든 위치를 알려 준다 — 이 안전망과 `default` 의 트레이드오프는 [[java17-lts-features]] · [[enum-constant-specific-polymorphism]].

### 블록 본문은 `yield`

```java
String type = switch (day) {
    case 6, 7 -> {
        log.info("주말 요청");
        yield "주말";          // switch 식의 결과값. 메서드는 계속 진행
    }
    default -> "평일";
};
// return "주말"; 로 쓰면 → error: attempt to return out of a switch expression
```

| 키워드 | 빠져나가는 범위 |
|---|---|
| `break` | switch **문**(또는 반복문)만. 값 없음 |
| `yield` | switch **식**에 값을 내놓고 빠져나감 |
| `return` | **메서드 전체** → switch 식 안에서는 금지 |

## 트레이드오프 / 한계

- 완전성 검사는 switch **식**에서만 강제된다. 값을 내놓지 않는 switch **문**은 빠진 경우가 있어도 컴파일된다.
- 블록 본문이 길어지면 switch 식이 읽기 어려워진다. 분기마다 로직이 크면 메서드로 추출하거나 enum 상수별 구현을 고려한다.

> [!WARNING]
> **오답 코너**
> - **"자바 14 에서 switch 가 fall-through 하지 않게 됐다"** — **화살표 형식만**이다. 콜론 형식은 그대로 fall-through 한다.
> - **"switch 에서 람다식을 쓸 수 있게 됐다"** — `->` 는 람다가 아닌 **case 규칙**이다. 객체를 만들지 않는다.
> - **"default 가 없으면 null 을 리턴한다"** — **컴파일 에러**(완전성 검사)다.
> - **"블록에서 값을 내는 키워드는 finish / return"** — **`yield`** 다. `return` 은 `attempt to return out of a switch expression` 컴파일 에러.

## 면접 답변

> "자바 14부터 switch 를 값을 내놓는 식으로 쓸 수 있게 됐습니다. 화살표 형식은 fall-through 가 없어서 break 를 쓰지 않아도 되고, 결과를 바로 변수에 대입할 수 있습니다. 또 식은 반드시 값을 내놓아야 하기 때문에 모든 경우를 다루지 않으면 컴파일 에러가 나고, 블록 안에서 값을 내놓을 때는 return 이 아니라 yield 를 씁니다."

## 복습 체크

- [ ] 문과 식의 차이를 한 문장으로 말하고, `if` 와 삼항 연산자로 예를 들 수 있는가?
- [ ] 옛 switch 로 값을 만들 때 왜 변수를 바깥에 선언해야 했는가?
- [ ] 화살표 `->` 가 람다가 아닌 이유는?
- [ ] 자바 14 이후에도 fall-through 가 일어나는 형식은?
- [ ] int switch 식에서 default 를 빼면? 왜 "없으면 null" 이 불가능한가?
- [ ] 블록 본문에서 값을 내놓는 키워드와, return 이 금지되는 이유는?

## 관련

[[switch-fall-through]] · [[java17-lts-features]] · [[enum-constant-specific-polymorphism]] · [[operator-evaluation-order]]
