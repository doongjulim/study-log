---
type: refined
slug: overriding-vs-overloading-binding
tags: [다형성, polymorphism, 서브타입다형성, subtype-polymorphism, 오버라이딩, overriding, 오버로딩, overloading, 동적바인딩, dynamic-binding, 동적디스패치, dynamic-dispatch, 정적바인딩, static-binding, 런타임, runtime, 컴파일타임, compile-time, 선언타입, declared-type, 정적타입, static-type, 실제클래스, runtime-type, 받는객체, receiver, 인자, argument, invokevirtual, invokestatic, vtable, 메서드시그니처, method-signature, method-hiding, static메서드]
topic: 다형성은 "하나의 호출 → 받는 객체의 실제 클래스 → 여러 구현"이며, 오버라이딩은 실행 시점 동적 바인딩, 오버로딩은 컴파일 시점 정적 바인딩이다
summary: 다형성은 하나의 호출 코드가 점(.) 왼쪽 받는 객체의 실제 클래스에 따라 여러 구현 중 하나로 실행되는 것이다. 오버라이딩은 실행 시점에 받는 객체의 실제 클래스를 보고 vtable 에서 몸통을 고르는 동적 바인딩이고, 오버로딩은 컴파일 시점에 인자의 선언 타입을 보고 시그니처를 바이트코드에 박는 정적 바인딩이다. 그래서 Object x = "hello"; print(x) 는 print(Object) 를 부르고, static 메서드는 invokestatic 으로 고정돼 오버라이딩(다형성)이 안 된다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-15.html#jls-15.12
  - https://docs.oracle.com/javase/specs/jvms/se17/html/jvms-6.html#jvms-6.5.invokevirtual
updated: 2026-10-09
---

# 다형성 — 오버라이딩(동적 바인딩) vs 오버로딩(정적 바인딩)

## 개념 정의

> **다형성**: **하나의 호출 코드**가, 점(.) 왼쪽의 **실제 객체**에 따라, **여러 구현** 중 하나로 실행되는 것.

| 칸 | `List` 예시 | enum 예시 ([[enum-constant-specific-polymorphism]]) |
|---|---|---|
| 하나의 호출 코드 | `list.add(0, "X")` | `searchType.matches(post, kw)` |
| 실제 객체 | ArrayList / LinkedList | TITLE / WRITER |
| 여러 구현 | 같은 배열 안 shift / 새 Node + first 연결 ([[arraylist-internals]], [[linkedlist-internals]]) | 제목 비교 / 작성자 비교 |

```java
list . add ( 0, "X" )
 ↑      ↑      ↑
 ①받는 객체  ②메서드 이름  ③인자
```

- 호출하는 쪽은 상대가 누군지 **모른 채** 같은 한 줄을 쓴다.
- 결정권은 ① **받는 객체**. ③ 인자(`"X"`)는 어느 구현이든 똑같이 받으므로 구현을 정할 수 없다.
- 개별 객체(상수 하나, ArrayList 하나)는 구현을 **1개**만 안다. "여러 개"인 것은 구현들의 집합이다.

## 작동 원리

### 오버라이딩 — 동적 바인딩

```java
SearchType searchType = ...;          // 선언 타입은 늘 SearchType
searchType.matches(post, keyword);
```

```
컴파일 시점:  "SearchType 타입에 matches 가 있나?" → 있음 → 통과 (어느 몸통인지는 모름)
            바이트코드: invokevirtual SearchType.matches
실행 시점:    searchType 이 가리키는 객체의 실제 클래스(SearchType$1 / $2) 확인
            → 그 클래스의 메서드 테이블(vtable)에서 재정의된 몸통 실행
```

### 오버로딩 — 정적 바인딩

```java
class Printer {
    void print(Object o) { System.out.println("Object 버전"); }
    void print(String s) { System.out.println("String 버전"); }
}

Object x = "hello";        // 실제 객체는 String
new Printer().print(x);    // → "Object 버전"
```

- 컴파일러는 코드를 **실행하지 않고 읽기만** 한다 → 아는 건 `x` 의 **선언 타입 `Object`** 뿐.
- `print(Object)` 와 `print(String)` 은 **이름만 같은 별개의 메서드** → 컴파일러가 시그니처를 확정해 박는다:

```
invokevirtual Printer.print(Ljava/lang/Object;)V    ← 컴파일 시점에 고정
```

### 비교표

| | 언제 | 무엇을 보고 | 바인딩 |
|---|---|---|---|
| **오버라이딩** | **실행 시점** | 점(.) **왼쪽 받는 객체의 실제 클래스** | 동적 |
| **오버로딩** | **컴파일 시점** | 괄호 안 **인자의 선언 타입** | 정적 |

### static 메서드는?

- static 메서드는 **오버라이딩되지 않는다.** 자식에 같은 시그니처 static 메서드를 만들면 **숨김(hiding)** 일 뿐.
- 호출은 `invokestatic Util.method` 로 **컴파일 시점에 고정** → 정적 바인딩 → **다형성 없음**. 그래서 구현 교체·테스트 대역이 필요한 기능은 static 이 아니라 빈으로 만든다 ([[static-util-vs-spring-bean]]).

## 트레이드오프 / 한계

- 동적 바인딩은 실행 시 테이블 조회 비용이 있지만, JIT 이 실제 타입이 하나뿐인 호출을 단일화·인라이닝해 대부분 상쇄한다 ([[interpreter-and-jit]]).
- 면접에서 "다형성" 은 보통 **오버라이딩 + 동적 바인딩**(받는 객체가 결정)을 가리킨다. 오버로딩을 "다형성" 이라고 부르는 분류(ad-hoc)도 있지만, 결정 시점·기준이 정반대라는 걸 함께 말해야 한다.
- 인자의 **실제 타입**으로도 분기하고 싶으면 이중 디스패치(Visitor 패턴)가 필요하다 — 자바는 받는 객체 하나만 동적으로 본다(single dispatch).

> [!WARNING]
> **오답 코너**
> - **"다형성 = 하나의 상수(객체)로부터 여러 메서드 동작"** — 객체 하나는 구현 **1개**만 안다. 하나인 것은 **호출 코드**, 여러 개인 것은 **구현**.
> - **"`list.add(0, "X")` 에서 구현을 정하는 건 `"X"`"** — `"X"` 는 **인자**다. 결정권은 점 **왼쪽 받는 객체**. 인자로 고르는 직관은 **오버로딩식 사고**다.
> - **"`Object x = "hello"; print(x)` 는 String 버전"** — 오버로딩은 컴파일 시점에 **선언 타입**(`Object`)으로 정해진다 → **Object 버전**.
> - **"오버라이딩은 변수의 값, 타입을 보고 결정"** — 변수의 **선언 타입은 늘 같다.** 결정을 바꾸는 건 변수가 **가리키는 객체의 실제 클래스**뿐.

## 복습 체크

- [ ] 다형성을 "하나 / 실제 / 여러" 세 칸으로 정의하고 `List` 예시를 채울 수 있는가?
- [ ] `list.add(0, "X")` 에서 받는 객체와 인자를 구분하고, 결정권이 어디 있는지 말할 수 있는가?
- [ ] `Object x = "hello"; print(x);` 의 출력과 그 이유는?
- [ ] 오버라이딩/오버로딩 각각 언제, 무엇을 보고 결정되는가?
- [ ] static 메서드가 다형성을 못 쓰는 이유를 바이트코드 명령으로 설명할 수 있는가?

## 관련

[[enum-constant-specific-polymorphism]] · [[static-util-vs-spring-bean]] · [[arraylist-internals]] · [[linkedlist-internals]] · [[interpreter-and-jit]] · [[java-optional]]
