---
date: 2026-10-23
type: raw
author: dongju
tags: [지역변수, local-variable, 인스턴스변수, instance-variable, 필드, field, 클래스변수, class-variable, static변수, static-field, 정적필드, 변수종류, variable-kinds, 생명주기, lifecycle, 스코프, scope, 공유상태, shared-state, 객체마다, 클래스에하나, 기본값, default-value, 초기화강제, definite-assignment, 확정대입, might-not-have-been-initialized, 컴파일러검사범위, 경로분석, flow-analysis, 메모리0채움, zeroing, fail-fast, final필드, final-field, 생성자끝까지, blank-final, 면접답변]
topic: 지역·인스턴스·클래스 변수는 개수와 수명이 다르고, 지역 변수만 초기화를 강제하는 이유는 컴파일러가 검사할 수 있는 범위라서다 — final 필드도 같은 원칙으로 강제된다
summary: 지역 변수는 메서드 호출마다, 인스턴스 변수는 객체마다, 클래스 변수는 클래스에 하나 존재해 모든 객체가 공유한다. 지역 변수의 일생은 메서드 하나의 중괄호 안에 다 있어 컴파일러가 모든 경로를 따라가 값 없이 읽히는 길을 찾을 수 있으므로 컴파일 에러로 초기화를 강제하고(definite assignment), 필드는 여러 메서드·클래스에서 실행 중에 정해지는 순서로 쓰여 검사할 수 없으므로 객체 생성 시 0으로 채운 기본값을 보장한다. 검사할 수 있는 곳에 0을 채우면 의도하지 않은 0이 조용히 흐르므로 fail-fast 를 택했다. final 필드는 모든 생성자가 끝나기 전에 한 번 대입돼야 하고 생성자는 검사 가능한 범위라 필드인데도 강제된다.
source: session
distilled: false
---

## 배운 개념

### 1. 세 변수의 차이

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
| 기본값 | ❌ | ⭕ | ⭕ |

- `count` 는 a·b 가 각자 한 칸씩 → 따로 셈. `total` 은 클래스에 한 칸 → a·b 가 같은 칸을 올림.
- 공유 칸이라 여러 스레드가 동시에 `inc()` 하면 갱신 유실 가능 ([[race-condition-lost-update]], [[singleton-bean-stateful]]).

### 2. 왜 지역 변수만 초기화를 강제하나

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
// 다른 클래스에서 acc.get() 을 init() 보다 먼저 부를지는 호출하는 쪽 마음
```

| | 지역 변수 `x` | 인스턴스 변수 `balance` |
|---|---|---|
| 일생이 들어 있는 곳 | 메서드 하나의 중괄호 안에 전부 | 여러 메서드·클래스에 흩어짐 |
| 읽고 쓰는 순서가 정해지는 때 | 코드에 이미 적혀 있음(if·for 경로) | **실행 중**에 호출하는 쪽이 정함 |
| 컴파일러 | 모든 경로를 따라가 검사 **가능** | 호출 순서를 미리 알 수 없어 검사 **불가능** |
| 자바의 선택 | 컴파일 에러로 **강제** | 객체 생성 시 메모리를 0으로 채워 **기본값 보장** |

> **컴파일러가 검사할 수 있으면 강제하고, 검사할 수 없으면 안전한 기본값을 보장한다.**

- 지역 변수도 0을 채워 주면 `flag` 가 false 일 때 **의도하지 않은 0이 조용히** 출력된다 → 컴파일 에러로 fail-fast (오버플로·switch 식 완전성 검사와 같은 축: [[integer-overflow]], [[switch-expression]]).

### 3. 원칙의 검증 — `final` 필드

```java
class Member {
    private final String name;   // 대입하지 않음
    public Member() { }
}
// Member.java:3: error: variable name might not have been initialized
//     public Member() { }
//                       ^          ← 생성자가 끝나는 지점을 가리킴

class Member2 {
    private final String name;
    public Member2(boolean flag) { if (flag) name = "DJ"; }   // flag=false 경로 → 같은 에러
}
```

- 필드인데도 기본값 null 로 넘어가지 않는다.
- 규칙: **모든 생성자가 끝나기 전에 정확히 한 번** 대입. 값은 실행 중(new 시) 들어가고, 컴파일러는 그 약속을 컴파일 때 검사한다.
- 검사 범위가 **생성자 본문**으로 좁혀지고, 생성자는 컴파일러가 전부 볼 수 있는 코드 → 지역 변수처럼 강제 가능. 원칙과 일치.

### 면접 답변

> "지역 변수는 메서드 호출마다, 인스턴스 변수는 객체마다, 클래스 변수는 클래스에 하나 존재합니다. 지역 변수만 초기화를 강제하는 건, 지역 변수는 메서드 하나 안에서 생기고 사라져서 컴파일러가 모든 경로를 검사할 수 있기 때문입니다. 필드는 어느 메서드가 어떤 순서로 호출될지 컴파일 시점에 알 수 없어서 대신 기본값을 보장합니다. 같은 이유로 final 필드는 생성자 안에서 검사가 가능해서 필드인데도 초기화가 강제됩니다."

## 오답·헷갈린 점

- 👍 `a.count = 2, b.count = 1, Counter.total = 3` — 정답.
- 👍 표(개수 ㉮㉯㉰ / 생성·소멸 ㉠㉡㉢ / 기본값 X·O·O) 전부 정답. (표를 직접 채우기 어렵다고 해 보기 고르기 방식으로 바꿔 진행)
- **전**: `total` 이 합쳐지는 이유 = "static 변수로써 **메모리에 올려서 관리**하기 때문" → **후**: `count` 도 힙 메모리에 있다. 차이는 **개수** — count 는 객체마다 한 칸, total 은 클래스에 한 칸이라 공유.
- 👍 (A) "조건문 안에 들어가지 않을 수도 있기 때문" / (B) "알 수 없다" — 정답. 단 **왜** 다른지(일생의 범위·순서가 정해지는 시점)는 보충받음.
- 👍 지역 변수에 0을 자동으로 채우지 않는 이유 = "의도하지 않은 값으로 조용히 돌아가기보단 컴파일 에러로 수정하도록" — 정답(fail-fast).
- **전**: `final` 필드는 "**컴파일 전까지** 값이 들어가야 한다" → **후**: 값은 실행 중에 들어간다. 규칙은 **모든 생성자가 끝나기 전에 한 번**, 그걸 컴파일러가 컴파일 때 **검사**. 실제 javac 에러가 생성자 닫는 중괄호를 가리키는 것으로 확인.

## Q&A

**Q. 지역·인스턴스·클래스 변수의 차이는?**
A. 개수(호출마다 / 객체마다 / 클래스에 하나), 수명(메서드 / 객체 / 클래스 로딩~종료), 기본값(X / O / O).

**Q. 왜 지역 변수만 초기화를 강제하나?**
A. 일생이 메서드 하나 안이라 컴파일러가 모든 경로를 검사할 수 있다. 필드는 호출 순서를 컴파일 때 알 수 없어 기본값으로 보장한다.

**Q. 그럼 지역 변수도 0으로 채워 주면?**
A. 의도하지 않은 0이 조용히 흐른다. 검사할 수 있으면 컴파일 에러가 낫다(fail-fast).

**Q. `final` 필드는 기본값을 받나?**
A. 아니다. 모든 생성자가 끝나기 전에 한 번 대입돼야 하고, 컴파일러가 생성자 경로를 검사해 강제한다.

## 다음 학습

- 인스턴스 초기화 순서: 필드 기본값 → 필드 초기화식·초기화 블록 → 생성자 본문
- static 공유 카운터를 `AtomicInteger` / `LongAdder` 로 바꾸는 이유 ([[race-condition-lost-update]] 복습)
