---
type: refined
slug: variable-kinds-initialization
tags: [지역변수, local-variable, 인스턴스변수, instance-variable, 필드, field, 클래스변수, class-variable, static변수, static-field, 정적필드, 변수종류, variable-kinds, 생명주기, lifecycle, 스코프, scope, 공유상태, shared-state, 객체마다, 클래스에하나, 기본값, default-value, 초기화강제, definite-assignment, 확정대입, might-not-have-been-initialized, 컴파일러검사범위, 경로분석, flow-analysis, 메모리0채움, zeroing, fail-fast, final필드, final-field, blank-final, 생성자끝까지, 면접답변]
topic: "지역·인스턴스·클래스 변수는 개수와 수명이 다르고, 지역 변수만 초기화를 강제하는 이유는 컴파일러가 검사할 수 있는 범위라서다 — final 필드도 같은 원칙으로 강제된다"
summary: "지역 변수는 메서드 호출마다, 인스턴스 변수는 객체마다, 클래스 변수는 클래스에 하나 존재해 모든 객체가 공유한다. 지역 변수의 일생은 메서드 하나의 중괄호 안에 다 있어 컴파일러가 모든 경로를 따라가 값 없이 읽히는 길을 찾을 수 있으므로 컴파일 에러로 초기화를 강제하고, 필드는 여러 메서드·클래스에서 실행 중에 정해지는 순서로 쓰여 검사할 수 없으므로 0으로 채운 기본값을 보장한다. 검사할 수 있는 곳에 0을 채우면 의도하지 않은 0이 조용히 흐르므로 fail-fast 를 택했다. final 필드는 모든 생성자가 끝나기 전에 한 번 대입돼야 하고 생성자는 검사 가능한 범위라 필드인데도 강제된다."
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-16.html
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-4.html#jls-4.12.5
updated: 2026-10-27
---

# 지역 · 인스턴스 · 클래스 변수와 초기화 규칙

## 개념 정의

```java
class Counter {
    static int total;   // 클래스 변수 (static 필드)
    int count;          // 인스턴스 변수 (필드)

    void inc() {
        int step = 1;   // 지역 변수
        count += step;
        total += step;
    }
}

Counter a = new Counter();
Counter b = new Counter();
a.inc(); a.inc(); b.inc();
// a.count = 2, b.count = 1, Counter.total = 3
```

| | 지역 변수 | 인스턴스 변수 (필드) | 클래스 변수 (static 필드) |
|---|---|---|---|
| 몇 개 | 메서드 **호출마다** 1개 | **객체마다** 1개 | **클래스에** 1개 (모든 객체 공유) |
| 생성 | 메서드 호출 시 | `new` 시 | 클래스 로딩 시(준비 단계: [[class-loading-lifecycle]]) |
| 소멸 | 메서드 끝 (스택 프레임 제거) | 객체가 GC 될 때 | 사실상 프로그램 끝 |
| 위치 | 스택 ([[jvm-stack-and-heap]]) | 힙(객체 안) | 클래스 단위 영역 |
| 기본값 | ❌ | ⭕ | ⭕ |

`count` 는 a·b 가 각자 한 칸씩이라 따로 세고, `total` 은 클래스에 한 칸이라 a·b 가 같은 칸을 올린다. 그래서 여러 스레드가 동시에 `inc()` 하면 `total` 은 갱신이 유실될 수 있다 ([[race-condition-lost-update]], [[singleton-bean-stateful]]).

## 작동 원리 — 왜 지역 변수만 초기화를 강제하나

```java
// (A) 지역 변수
void print(boolean flag) {
    int x;
    if (flag) {
        x = 10;
    }
    System.out.println(x);   // ❌ variable x might not have been initialized
}

// (B) 인스턴스 변수
class Account {
    int balance;
    void init() { balance = 100; }
    int  get()  { return balance; }
}
// 다른 클래스에서 get() 을 init() 보다 먼저 부를지는 호출하는 쪽 마음
```

| | 지역 변수 `x` | 인스턴스 변수 `balance` |
|---|---|---|
| 일생이 들어 있는 곳 | 메서드 하나의 중괄호 안에 전부 | 여러 메서드·클래스에 흩어짐 |
| 읽고 쓰는 순서가 정해지는 때 | 코드에 이미 적혀 있음(if·for 경로) | **실행 중**에 호출하는 쪽이 정함 |
| 컴파일러 | 모든 경로를 따라가 검사 **가능** | 호출 순서를 미리 알 수 없어 검사 **불가능** |
| 자바의 선택 | 컴파일 에러로 **강제** (definite assignment) | 객체 생성 시 메모리를 0으로 채워 **기본값 보장** |

> **컴파일러가 검사할 수 있으면 강제하고, 검사할 수 없으면 안전한 기본값을 보장한다.**

지역 변수에도 0을 채워 주면 `flag` 가 false 일 때 **의도하지 않은 0이 조용히** 출력된다. 검사할 수 있는 곳에서는 컴파일 에러라는 가장 빠른 fail-fast 를 택했다 ([[integer-overflow]], [[switch-expression]] 과 같은 축).

## 트레이드오프 / 한계 — 원칙으로 설명되는 예외들

### `final` 필드는 필드인데도 강제된다

```java
class Member {
    private final String name;   // 대입하지 않음
    public Member() { }
}
// error: variable name might not have been initialized
//     public Member() { }
//                       ^          ← 생성자가 끝나는 지점을 가리킴

class Member2 {
    private final String name;
    public Member2(boolean flag) { if (flag) name = "DJ"; }   // flag=false 경로 → 같은 에러
}
```

- 규칙: **모든 생성자가 끝나기 전에 정확히 한 번** 대입. 값은 실행 중(new 시) 들어가고, 컴파일러는 그 약속을 컴파일 때 **검사**한다.
- 검사 범위가 **생성자 본문**으로 좁혀지고, 생성자는 컴파일러가 전부 볼 수 있는 코드라 지역 변수처럼 강제할 수 있다.

### 람다는 지역 변수를 바꿀 수 없다

지역 변수는 메서드와 함께 사라지므로 람다는 값을 복사해 캡처하고, 그래서 effectively final 만 허용된다 ([[lambda-capture-effectively-final]]).

> [!WARNING]
> **오답 코너**
> - **"static 변수는 메모리에 올려서 관리하기 때문에 합쳐진다"** — 인스턴스 변수도 힙 메모리에 있다. 차이는 **개수**다. 객체마다 한 칸이냐, 클래스에 한 칸이냐.
> - **"필드는 검사할 수 없다"(이유 없이)** — 이유까지: 필드는 여러 메서드에서 **실행 중에 정해지는 순서**로 쓰인다. 지역 변수는 일생이 메서드 하나 안에 다 있다.
> - **"필드 기본값은 기본 생성자가 채워 준다 (`name = new String()`)"** — 기본값은 `new` 로 힙 메모리를 할당하는 순간 채워진다. 생성자 실행 **전**이고, 참조형은 `""` 가 아니라 `null` ([[default-constructor]]).
> - **"final 필드는 컴파일 전까지 값이 들어가야 한다"** — 값은 실행 중에 들어간다. 규칙은 **모든 생성자가 끝나기 전에 한 번**이고, 그걸 컴파일러가 컴파일 때 검사한다.

## 면접 답변

> "지역 변수는 메서드 호출마다, 인스턴스 변수는 객체마다, 클래스 변수는 클래스에 하나 존재합니다. 지역 변수만 초기화를 강제하는 건, 지역 변수는 메서드 하나 안에서 생기고 사라져서 컴파일러가 모든 경로를 검사할 수 있기 때문입니다. 필드는 어느 메서드가 어떤 순서로 호출될지 컴파일 시점에 알 수 없어서 대신 기본값을 보장합니다. 같은 이유로 final 필드는 생성자 안에서 검사가 가능해서 필드인데도 초기화가 강제됩니다."

## 복습 체크

- [ ] `Counter` 예제의 세 출력값과, `count` 와 `total` 이 다른 이유를 "개수"로 설명할 수 있는가?
- [ ] 세 변수의 생성·소멸 시점과 기본값 여부를 말할 수 있는가?
- [ ] 지역 변수만 초기화를 강제하는 이유를 "컴파일러가 검사할 수 있는 범위"로 설명할 수 있는가?
- [ ] 지역 변수에 0을 자동으로 채우지 않는 이유는?
- [ ] `final` 필드가 필드인데도 초기화를 강제받는 이유와, 그 기한은?

## 관련

[[class-loading-lifecycle]] · [[default-constructor]] · [[primitive-vs-reference-type]] · [[jvm-stack-and-heap]] · [[lambda-capture-effectively-final]] · [[race-condition-lost-update]] · [[singleton-bean-stateful]] · [[java-array]]
