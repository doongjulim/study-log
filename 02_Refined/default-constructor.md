---
type: refined
slug: default-constructor
tags: [기본생성자, 기본 생성자, default-constructor, no-arg-constructor, 매개변수없는생성자, 생성자, constructor, 컴파일러, javac, compiler, 자동생성, 생성조건, 생성자가하나도없을때, "super()", 부모생성자, Object, 접근제어자, 생성자오버로딩, constructor-overloading, 선언vs호출, declaration-vs-invocation, 객체생성순서, new실행순서, 필드기본값, default-value, null, "cannot be applied to given types", JPA엔티티, NoArgsConstructor, 면접답변]
topic: 기본 생성자는 컴파일러가 "클래스에 생성자가 하나도 없을 때만" public Dog() { super(); } 로 넣어 준다 — 호출부가 아니라 선언부를 보고 결정한다
summary: 기본 생성자는 JVM 이 아니라 컴파일러(javac)가 컴파일할 때 추가하며, 매개변수 없이 본문은 super(); 한 줄이고 접근제어자는 클래스를 따른다. 필드의 null·0 은 생성자가 아니라 new 로 힙 메모리를 할당하는 순간 채워지는 기본값이라 기본 생성자는 필드를 건드리지 않는다. 컴파일러는 호출하는 쪽의 인자 유무가 아니라 클래스 선언부만 보고, 생성자가 하나라도 선언돼 있으면 아무것도 추가하지 않는다 — Dog(String) 만 있으면 new Dog() 는 컴파일 에러이고, 필요하면 Dog() {} 를 직접 오버로딩해야 한다. 인자 있는 생성자를 둔 JPA 엔티티에 기본 생성자를 직접 추가해야 하는 이유도 이것이다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-8.html#jls-8.8.9
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-12.html#jls-12.5
updated: 2026-10-27
---

# 기본 생성자 (default constructor)

## 개념 정의

> 컴파일러(javac)는 클래스에 **생성자가 하나도 선언돼 있지 않을 때만** 기본 생성자를 추가한다.

```java
public class Dog {
    String name;
    int age;

    public Dog() {      // ← 컴파일러가 추가
        super();        // ← 부모(Object)의 생성자 호출
    }
}
```

- **누가·언제**: 컴파일러가 `.java` → `.class` 로 만들 때. JVM 은 이미 만들어진 `.class` 를 실행할 뿐이다.
- **매개변수 없음** — 그래서 "기본" 생성자.
- **접근제어자는 클래스를 따른다** — `public class` → `public Dog()`.
- **본문은 `super();` 한 줄.** 모든 생성자의 첫 줄은 `super(...)` 또는 `this(...)` 여야 하고, 안 쓰면 컴파일러가 `super();` 를 넣는다 — 직접 작성한 생성자에도 해당.
- **필드를 초기화하지 않는다.**

## 작동 원리

### `new Dog()` 실행 순서

1. 힙에 메모리 할당 → 필드에 타입별 **기본값** (참조형 `null`, `int` `0`, `boolean` `false`, `double` `0.0`)
2. 필드 명시적 초기화·초기화 블록 (있다면)
3. 생성자 호출 → 첫 줄 `super()` 로 `Object()` 실행 → 본문 실행 (기본 생성자는 할 일 없음)
4. 참조값 반환 → 변수에 저장 ([[class-and-instance]])

```java
Dog b = new Dog();
System.out.println(b.name);           // null  ("" 아님)
System.out.println(b.age);            // 0
System.out.println(b.name.length());  // NullPointerException
```

- 필드가 초기화 없이도 컴파일되는 건 생성자 덕분이 아니라 **할당 시점의 기본값 보장** 덕분이다 ([[variable-kinds-initialization]]).

### 생성 조건 — 선언부를 본다

| `Dog` 클래스 상태 | 컴파일러 | `new Dog()` |
|---|---|---|
| 생성자를 하나도 안 씀 | `Dog()` 추가 | ✅ |
| `Dog(String name)` 을 직접 씀 | **아무것도 추가 안 함** | ❌ 컴파일 에러 |

```java
public class Dog {
    String name;
    Dog(String name) { this.name = name; }
}

Dog a = new Dog("초코");  // ① 된다 — Dog(String) 이 선언돼 있음
Dog b = new Dog();        // ② 컴파일 에러
```

```
error: constructor Dog in class Dog cannot be applied to given types;
  required: String
  found:    no arguments
```

- **왜 이렇게 설계했나**: `Dog(String name)` 을 만들었다는 건 "이름이 있어야 Dog 를 만들 수 있다" 는 의도. 컴파일러가 `Dog()` 를 끼워 넣으면 그 의도(불변식)가 깨진다.

### ②를 되게 하려면 — 생성자 오버로딩

```java
public class Dog {
    String name;
    Dog(String name) { this.name = name; }
    Dog() {}                       // 매개변수 없는 생성자를 직접 선언
}
```

- 같은 클래스에 매개변수가 다른 생성자를 하나 더 두는 것 = **오버로딩**. 생성자는 상속되지 않으므로 오버라이딩 대상이 아니고 `@Override` 는 컴파일 에러 ([[overriding-vs-overloading-binding]]).
- 고칠 곳은 호출부(`new Dog()`)가 아니라 **클래스 안의 선언**이다.

## 트레이드오프 / 한계

- 인자 있는 생성자를 추가하는 순간 기본 생성자가 **조용히 사라진다.** 같은 소스에서 `new Dog()` 를 쓰면 컴파일 에러로 바로 드러나지만, **리플렉션으로 기본 생성자를 찾는 프레임워크**(JPA/Hibernate, 일부 직렬화 라이브러리)는 컴파일은 통과하고 **런타임에** 실패한다 → JPA 엔티티는 기본 생성자를 직접 둔다 ([[jpa-entity-default-constructor]]).
- 부모 클래스에 기본 생성자가 없으면, 자식 생성자에 자동 삽입된 `super();` 가 호출할 대상이 없어 자식 쪽이 컴파일 에러가 난다 → 자식 생성자 첫 줄에 `super(인자)` 를 직접 써야 한다.

> [!WARNING]
> **오답 코너**
> - **"자바가 알아서 만든다"** — 주체는 **컴파일러(javac)**, 시점은 **컴파일**. JVM 이 아니다.
> - **"기본 생성자가 `name = new String()` 으로 초기화하고 int 는 생략된다"** — `new String()` 이면 `""` 라 `b.name.length()` 가 0 이어야 하지만 실제론 NPE. `name` 은 `null`, `age` 는 `0` 이고, 이는 생성자가 아니라 **할당 시점의 기본값**. 기본 생성자 본문은 `super();` 뿐.
> - **"호출할 때 인자가 없으면(매개변수 값이 없으면) 기본 생성자가 만들어진다"** — 컴파일러는 **호출부가 아니라 선언부**를 본다. 생성자가 하나라도 있으면 추가하지 않는다. `Dog(String)` 만 있을 때 `new Dog("초코")` 는 되고 `new Dog()` 는 에러 — 정반대로 답하기 쉽다.
> - **"매개변수 없는 생성자를 추가하려면 `@Override`"** — 생성자 추가는 **오버로딩**. 생성자는 상속되지 않아 오버라이딩 불가.
> - **"생성자 추가 = `Dog dog = new Dog();` 작성"** — 그건 **호출**. 필요한 건 클래스 안의 **선언** `Dog() {}`.

## 면접 답변

> "기본 생성자는 컴파일러가 컴파일할 때, 클래스에 생성자가 하나도 선언돼 있지 않은 경우에만 매개변수 없이 `super()` 만 호출하는 형태로 추가합니다. 개발자가 인자 있는 생성자를 하나라도 선언하면 그 클래스는 그 인자로만 만들라는 의도로 보고 추가하지 않습니다. 그래서 JPA 엔티티처럼 리플렉션으로 기본 생성자를 호출하는 프레임워크를 쓸 때는, 인자 있는 생성자를 두면 기본 생성자를 직접 선언해야 하고 보통 protected 로 둡니다. 참고로 필드의 null이나 0은 생성자가 아니라 객체 메모리를 할당하는 시점에 채워지는 기본값입니다."

## 복습 체크

- [ ] 기본 생성자를 **누가, 언제, 어떤 조건에서** 만드는지 한 문장으로 말할 수 있는가?
- [ ] 컴파일러가 넣는 기본 생성자를 코드로 쓸 수 있는가? 본문엔 무엇이 있는가?
- [ ] 생성자 없는 클래스에서 `new Dog()` 후 `b.name`, `b.age`, `b.name.length()` 의 결과와 이유는?
- [ ] `Dog(String)` 만 있을 때 `new Dog("초코")`, `new Dog()` 는 각각? 고치는 방법은?
- [ ] 생성자 추가가 오버로딩인 이유, `@Override` 를 못 붙이는 이유는?
- [ ] 인자 있는 생성자만 있는 JPA 엔티티가 컴파일은 되는데 조회 시 실패하는 이유는?

## 관련

[[class-and-instance]] · [[jpa-entity-default-constructor]] · [[overriding-vs-overloading-binding]] · [[variable-kinds-initialization]] · [[method-signature]] · [[java-record]]
