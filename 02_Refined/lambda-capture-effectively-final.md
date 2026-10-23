---
type: refined
slug: lambda-capture-effectively-final
tags: [람다, lambda, 람다캡처, lambda-capture, 변수캡처, variable-capture, effectively-final, 사실상final, final, 지역변수캡처, 값복사, value-copy, 스택프레임, stack-frame, 메서드종료, 다른스레드, this캡처, 필드캡처, 인스턴스변수, 공유상태, 동시성, race-condition, forEach, "count++", must-be-final-or-effectively-final, 면접답변]
topic: "람다는 지역 변수를 값으로 복사해 캡처하므로 effectively final 만 허용한다 — 필드는 this 를 캡처하고 힙에 살아 있어 허용되지만 동시성 위험은 남는다"
summary: "지역 변수는 메서드 스택 프레임과 함께 사라지는데 람다는 메서드가 끝난 뒤 다른 스레드에서 실행될 수도 있다. 그래서 자바는 람다를 만들 때 지역 변수의 값을 복사해 람다 객체 안에 넣어 두고(캡처), 사본을 바꾸게 두면 원본과 어긋나 개발자를 속이므로 바뀌지 않는 effectively final 변수만 캡처하게 한다. 필드는 람다가 this 참조를 캡처하고 필드 자체는 힙의 객체 안에 살아 있어 바꿀 수 있지만, 여러 스레드가 동시에 바꾸면 갱신 유실이 날 수 있다."
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-15.html#jls-15.27.2
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-4.html#jls-4.12.4
updated: 2026-10-23
---

# 람다 캡처와 effectively final

## 개념 정의

```java
int count = 0;
List.of("a", "b", "c").forEach(s -> count++);   // ❌
// error: local variables referenced from a lambda expression must be final or effectively final
```

- `count` 를 **인스턴스 변수(필드)** 로 바꾸면 컴파일된다.
- **effectively final**: `final` 을 붙이지 않았어도 선언 후 한 번도 재대입하지 않은 변수.

## 작동 원리 — 람다는 지역 변수의 값을 복사한다

- 지역 변수는 메서드 안에서만 살아 있고, 메서드가 끝나면 스택 프레임과 함께 사라진다 ([[variable-kinds-initialization]]).
- 람다는 메서드가 끝난 **뒤에**, **다른 스레드**에서 실행될 수도 있다 → 원본이 남아 있으리라는 보장이 없다.
- 그래서 람다를 만들 때 값을 **복사**해 람다 객체 안에 넣어 둔다(캡처).

```
메서드 스택 프레임: count = 0  ──(값 복사)──▶  람다 객체 안: count 사본 = 0
메서드 끝 → 프레임 사라짐                         람다는 사본으로 계속 실행 가능
```

- 사본이니 `count++` 를 허용하면 **원본과 사본이 따로 논다.** 개발자는 원본이 바뀐다고 착각한다 → 아예 **바뀌지 않는 변수만** 캡처 가능하게 막았다.

### 필드는 왜 허용되나 — `this` 를 캡처한다

- 람다는 필드 값이 아니라 **`this`(객체 참조)** 를 캡처한다. 필드는 힙의 객체 안에 살아 있어 메서드가 끝나도 남는다.
- 그래서 람다 안에서 필드를 바꿀 수 있다.

## 트레이드오프 / 한계

- 필드 변경이 **허용된다고 안전한 건 아니다.** 공유 상태라 여러 스레드가 동시에 바꾸면 갱신 유실이 난다 ([[race-condition-lost-update]]).
- 람다 안에서 바깥 상태를 바꾸는 것 자체가 Stream 의 안티패턴이다. 집계는 `count()`·`collect`·`reduce` 로 결과를 돌려받는다 ([[stream-vs-for-loop]]).
- `int[] box = {0};` 처럼 배열로 감싸 "참조는 final, 내용은 변경"하는 우회는 컴파일만 통과시킬 뿐 동시성 문제를 그대로 가진다. `AtomicInteger` 는 원자성은 해결하지만, 람다로 바깥 상태를 바꾸는 구조라는 점은 같다.

> [!WARNING]
> **오답 코너**
> - **"람다는 지역 변수를 참조로 들고 있다"** — **값을 복사**해 캡처한다. 그래서 바꾸지 못하게 막는다.
> - **"필드는 바꿀 수 있으니 안전하다"** — 컴파일만 될 뿐 공유 상태의 동시성 위험은 그대로다.

## 면접 답변

> "람다는 메서드가 끝난 뒤 다른 스레드에서 실행될 수도 있는데, 지역 변수는 메서드가 끝나면 스택에서 사라집니다. 그래서 람다는 지역 변수의 값을 복사해서 가지고 있고, 복사본을 바꾸면 원본과 어긋나기 때문에 바뀌지 않는 effectively final 변수만 쓸 수 있게 막습니다. 필드는 람다가 객체 참조를 잡고 있어서 바꿀 수 있지만, 여러 스레드가 동시에 바꾸면 동시성 문제가 생깁니다."

## 복습 체크

- [ ] 람다 안에서 지역 변수를 바꿀 수 없는 이유를 "스택 프레임"과 "값 복사"로 설명할 수 있는가?
- [ ] effectively final 의 정의는?
- [ ] 필드는 왜 람다 안에서 바꿀 수 있고, 그래도 남는 위험은?
- [ ] forEach 로 카운트하는 대신 무엇을 써야 하는가?

## 관련

[[variable-kinds-initialization]] · [[stream-vs-for-loop]] · [[race-condition-lost-update]] · [[lambda-checked-exception]] · [[jvm-stack-and-heap]]
