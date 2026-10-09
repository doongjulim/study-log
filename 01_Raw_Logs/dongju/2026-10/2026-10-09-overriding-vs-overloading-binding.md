---
date: 2026-10-09
type: raw
author: dongju
tags: [오버라이딩, overriding, 오버로딩, overloading, 동적 바인딩, dynamic binding, 동적 디스패치, dynamic dispatch, 정적 바인딩, static binding, 런타임, runtime, 컴파일 타임, compile time, 선언 타입, declared type, 정적 타입, static type, 실제 클래스, runtime type, 받는 객체, receiver, 인자, argument, invokevirtual, vtable, 메서드 시그니처, method signature, 다형성, polymorphism]
topic: 오버라이딩과 오버로딩은 "언제, 무엇을 보고" 구현을 고르는가 — 동적 바인딩(받는 객체의 실제 클래스) vs 정적 바인딩(인자의 선언 타입)
summary: 오버라이딩은 실행 시점에 점(.) 왼쪽 받는 객체의 실제 클래스를 보고 몸통을 고르는 동적 바인딩이고, 오버로딩은 컴파일 시점에 괄호 안 인자의 선언 타입을 보고 시그니처를 바이트코드에 박는 정적 바인딩이다. 그래서 Object x = "hello"; print(x) 는 실제 객체가 String 이어도 print(Object) 가 호출된다. 면접에서 말하는 다형성은 보통 오버라이딩 + 동적 바인딩 쪽이다.
source: session
distilled: false
---

## 배운 개념

### 1. 오버로딩 함정

```java
class Printer {
    void print(Object o) { System.out.println("Object 버전"); }
    void print(String s) { System.out.println("String 버전"); }
}

Object x = "hello";        // 실제 객체는 String
new Printer().print(x);    // → "Object 버전"
```

- 컴파일러는 코드를 **실행하지 않고 읽기만** 한다 → 아는 건 `x` 의 **선언 타입 `Object`** 뿐.
- `print(Object)` 와 `print(String)` 은 **이름만 같은 별개의 메서드** → 컴파일러가 바이트코드에 **시그니처까지 확정**해서 넣어야 한다:

```
invokevirtual Printer.print(Ljava/lang/Object;)V    ← 컴파일 시점에 고정
```

### 2. 오버라이딩은 반대

```java
SearchType searchType = ...;          // 선언 타입은 늘 SearchType
searchType.matches(post, keyword);    // 바이트코드: invokevirtual SearchType.matches
```

실행 시 JVM 이 `searchType` 이 **가리키는 객체의 실제 클래스**(`SearchType$1` / `$2`)를 확인하고 그 클래스의 메서드 테이블(vtable)에서 재정의된 몸통을 찾는다 ([[enum-constant-specific-polymorphism]]).

### 3. 비교표

| | 언제 | 무엇을 보고 | 바인딩 |
|---|---|---|---|
| **오버라이딩** | **실행 시점** | 점(.) **왼쪽 받는 객체의 실제 클래스** | 동적 |
| **오버로딩** | **컴파일 시점** | 괄호 안 **인자의 선언 타입** | 정적 |

```java
list . add ( 0, "X" )
 ↑             ↑
 오버라이딩이 보는 곳     오버로딩이 보는 곳 (단, 선언 타입만)
```

- 면접에서 "다형성" 은 보통 **오버라이딩 + 동적 바인딩** (받는 객체가 결정하는 쪽).

## 오답·헷갈린 점

- **전**: `print(x)` 는 **"String 버전"** — 호출 시 변수의 값을 보고 결정 → **후**: **"Object 버전"**. 오버로딩은 **컴파일 시점, 인자의 선언 타입**으로 결정. 힌트(컴파일러가 아는 것 / 바이트코드 시그니처) 후 정정.
- **전**: 오버라이딩은 호출 시 "변수 **값, 타입**" 을 보고 결정 → **후**: 변수의 **선언 타입은 늘 같다**(SearchType). 결정을 바꾸는 건 변수가 **가리키는 객체의 실제 클래스**뿐. 선언 타입만 보는 건 오버로딩.
- **연결 고리**: 같은 세션 초반에 `list.add(0, "X")` 에서 `"X"`(인자)가 구현을 정한다고 답한 직관 = **오버로딩식 사고**. 다형성(오버라이딩)의 결정권은 인자가 아니라 받는 객체.

## Q&A

- **Q. `Object x = "hello"; print(x);` 의 출력은?** A. "Object 버전".
- **Q. 오버라이딩/오버로딩은 각각 언제, 무엇을 보고 결정?** A. 실행 시점·받는 객체의 실제 클래스 / 컴파일 시점·인자의 선언 타입.
- **Q. 컴파일러가 오버로딩을 미리 정해야 하는 이유?** A. 둘은 별개 메서드라 바이트코드에 시그니처를 확정해서 써야 하기 때문.

## 다음 학습

- 오버로딩 해석 우선순위: 정확 일치 → 확대 변환(widening) → 박싱 → 가변인자
- `print(null)` 처럼 모호한 호출의 컴파일 에러
- 이중 디스패치(Double Dispatch)와 Visitor 패턴 — 인자의 실제 타입으로도 분기하고 싶을 때
