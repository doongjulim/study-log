---
date: 2026-10-23
type: raw
author: dongju
tags: [람다, lambda, 람다캡처, lambda-capture, 변수캡처, variable-capture, effectively-final, 사실상final, final, 지역변수캡처, 값복사, value-copy, 스택프레임, stack-frame, 메서드종료, 다른스레드, this캡처, 필드캡처, 인스턴스변수, 공유상태, 동시성, race-condition, forEach, count++, local-variables-referenced-from-a-lambda-expression-must-be-final-or-effectively-final, 면접답변]
topic: 람다는 지역 변수를 값으로 복사해 캡처하므로 effectively final 만 허용한다 — 필드는 this 를 캡처하고 힙에 살아 있어 허용되지만 동시성 위험은 남는다
summary: 지역 변수는 메서드 스택 프레임과 함께 사라지는데 람다는 메서드가 끝난 뒤 다른 스레드에서 실행될 수도 있다. 그래서 자바는 람다를 만들 때 지역 변수의 값을 복사해 람다 객체 안에 넣어 두고(캡처), 사본을 바꾸게 두면 원본과 어긋나 개발자를 속이므로 바뀌지 않는 effectively final 변수만 캡처하게 한다. 필드는 람다가 this 참조를 캡처하고 필드 자체는 힙의 객체 안에 살아 있어 람다 안에서 바꿀 수 있지만, 여러 스레드가 동시에 바꾸면 갱신 유실이 날 수 있다.
source: session
distilled: false
---

## 배운 개념

### 1. 지역 변수는 람다 안에서 바꿀 수 없다

```java
int count = 0;
List.of("a", "b", "c").forEach(s -> count++);   // ❌
// error: local variables referenced from a lambda expression must be final or effectively final
```

- `count` 를 **인스턴스 변수(필드)** 로 바꾸면 컴파일된다.

### 2. 이유 — 람다는 지역 변수의 값을 복사해 캡처한다

- 지역 변수는 메서드 안에서만 살아 있고, 메서드가 끝나면 스택 프레임과 함께 사라진다.
- 람다는 메서드가 끝난 **뒤에**, **다른 스레드**에서 실행될 수도 있다 → 원본 `count` 가 남아 있을지 모른다.
- 그래서 람다를 만들 때 값을 **복사**해 람다 객체 안에 넣어 둔다(캡처).

```
메서드 스택 프레임: count = 0  ──(값 복사)──▶  람다 객체 안: count 사본 = 0
메서드 끝 → 프레임 사라짐                         람다는 사본으로 계속 실행 가능
```

- 사본이니 `count++` 를 허용하면 **원본과 사본이 따로 논다** → 개발자는 원본이 바뀐다고 착각 → 아예 **바뀌지 않는 변수(effectively final)만** 캡처 가능.
- effectively final = `final` 을 안 붙였어도 선언 후 한 번도 재대입하지 않은 변수.

### 3. 필드는 왜 허용되나 — `this` 를 캡처한다

- 람다는 필드 값이 아니라 **`this`(객체 참조)** 를 캡처한다. 필드는 힙의 객체 안에 살아 있어 메서드가 끝나도 남는다.
- 그래서 람다 안에서 필드를 바꿀 수 있다. 단 **공유 상태**라 여러 스레드가 동시에 바꾸면 갱신 유실 ([[race-condition-lost-update]]). Stream 에서 forEach 로 공유 상태를 바꾸는 것 자체가 안티패턴 ([[stream-vs-for-loop]]).

### 면접 답변

> "람다는 메서드가 끝난 뒤 다른 스레드에서 실행될 수도 있는데, 지역 변수는 메서드가 끝나면 스택에서 사라집니다. 그래서 람다는 지역 변수의 값을 복사해서 가지고 있고, 복사본을 바꾸면 원본과 어긋나기 때문에 바뀌지 않는 effectively final 변수만 쓸 수 있게 막습니다. 필드는 람다가 객체 참조를 잡고 있어서 바꿀 수 있지만, 여러 스레드가 동시에 바꾸면 동시성 문제가 생깁니다."

## 오답·헷갈린 점

- 👍 "지역 변수는 메서드 안에서만 살아 있고 메서드가 끝남과 동시에 지워지는데, 람다는 다른 스레드에서도 실행될 수 있어 count 가 남아 있을지 몰라 사전에 차단된다" — 정답.
- 보충: 자바의 실제 해결 방식(값 **복사** 캡처)과, 그래서 "바꾸지 못하게" 막는 이유(사본과 원본의 불일치), 필드가 허용되는 이유(`this` 캡처 + 힙)와 남는 동시성 위험.

## Q&A

**Q. 람다 안에서 지역 변수를 바꿀 수 없는 이유는?**
A. 람다는 지역 변수 값을 복사해 캡처한다. 지역 변수는 메서드가 끝나면 사라지고, 사본을 바꾸게 하면 원본과 어긋나므로 effectively final 만 허용한다.

**Q. 필드는 왜 바꿀 수 있나?**
A. 람다가 this 를 캡처하고 필드는 힙에 살아 있기 때문. 대신 공유 상태라 동시성 위험이 있다.

## 다음 학습

- 람다 vs 익명 클래스: `this` 가 가리키는 대상, 캡처 방식 차이
- forEach 공유 상태 대신 collect / reduce 로 바꾸기 ([[stream-vs-for-loop]] 복습)
