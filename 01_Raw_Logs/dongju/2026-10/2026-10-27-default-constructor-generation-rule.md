---
date: 2026-10-27
type: raw
author: dongju
tags: [기본 생성자, default constructor, no-arg constructor, 생성자, constructor, 컴파일러, javac, compiler, 자동생성, 생성규칙, "super()", Object, 생성자 오버로딩, constructor overloading, 오버라이딩, overriding, "@Override", 선언 vs 호출, declaration vs invocation, 필드 기본값, default value, null, 객체 생성 순서, new 실행 순서, JPA 엔티티, NoArgsConstructor, AccessLevel.PROTECTED, "@Getter", 면접답변]
topic: 기본 생성자는 컴파일러가 컴파일할 때, 클래스에 생성자가 하나도 없을 때만 public Dog() { super(); } 형태로 추가한다 — 호출부가 아니라 선언부를 보고 결정한다
summary: 기본 생성자는 JVM 이 아니라 컴파일러(javac)가 .java 를 .class 로 만들 때 넣어 주며, 모양은 매개변수 없고 본문은 super(); 한 줄, 접근제어자는 클래스를 따른다. 필드의 null·0 은 생성자가 아니라 new 로 힙 메모리가 할당되는 순간 채워지는 타입별 기본값이다. 컴파일러는 호출하는 쪽이 아니라 클래스 선언부만 보고, 생성자가 하나라도 선언돼 있으면 기본 생성자를 추가하지 않는다 — Dog(String) 만 있으면 new Dog() 는 컴파일 에러. 그래서 인자 있는 생성자를 둔 JPA 엔티티에는 기본 생성자를 직접(protected) 추가해야 한다.
source: session
distilled: true
distilled_at: 2026-10-27
---

## 배운 개념

### 1. 누가, 언제 — 컴파일러가 컴파일할 때

- 넣어 주는 주체는 **컴파일러(javac)**. `.java` → `.class` 로 만들 때 추가한다. JVM 은 이미 만들어진 `.class` 를 실행할 뿐이다.

### 2. 기본 생성자의 실제 모양

```java
public class Dog {
    String name;
    int age;

    public Dog() {      // ← 컴파일러가 추가
        super();        // ← 부모(Object)의 생성자 호출
    }
}
```

- 매개변수 없음 (그래서 "기본" 생성자)
- 접근제어자는 클래스를 따른다 (`public class` → `public Dog()`)
- 본문은 `super();` 한 줄. 모든 생성자의 첫 줄은 `super(...)` 또는 `this(...)` 여야 하고, 안 쓰면 컴파일러가 `super();` 를 넣는다 — 직접 작성한 생성자에도 해당.
- **필드를 초기화하지 않는다.** 필드 기본값은 생성자와 무관하다.

### 3. `new Dog()` 실행 순서

1. 힙에 메모리 할당 → 필드에 타입별 기본값 (참조형 `null`, `int` `0`, `boolean` `false`, `double` `0.0`)
2. `Dog()` 생성자 호출 → 첫 줄 `super()` 로 `Object()` 실행
3. `Dog()` 본문 실행 (기본 생성자는 할 일 없음)
4. 참조값 반환 → 변수에 저장

```java
Dog b = new Dog();
System.out.println(b.name);           // null  ("" 아님)
System.out.println(b.age);            // 0
System.out.println(b.name.length());  // NullPointerException
```

### 4. 생성 조건 — "생성자가 하나도 없을 때만"

> 컴파일러는 클래스에 **생성자가 하나도 없을 때만** 기본 생성자를 만들어 준다. 호출부가 아니라 **선언부**를 보고 결정한다.

| `Dog` 클래스 상태 | 컴파일러 | `new Dog()` |
|---|---|---|
| 생성자를 하나도 안 씀 | `Dog()` 추가 | ✅ |
| `Dog(String name)` 을 직접 씀 | 아무것도 추가 안 함 | ❌ 컴파일 에러 |

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

- 이렇게 설계한 이유: `Dog(String name)` 을 만들었다는 건 "이름이 있어야 Dog 를 만들 수 있다" 는 의도. 컴파일러가 `Dog()` 를 끼워 넣으면 그 의도가 깨진다.

### 5. ②를 되게 하려면 — 생성자 오버로딩

```java
public class Dog {
    String name;
    Dog(String name) { this.name = name; }
    Dog() {}                               // 매개변수 없는 생성자를 직접 선언
}
```

- 같은 클래스에 매개변수가 다른 생성자를 하나 더 두는 것 = **오버로딩**.
- 생성자는 상속되지 않으므로 오버라이딩 대상이 아니고, `@Override` 를 붙이면 컴파일 에러.

### 6. 실무 연결 — JPA 엔티티

```java
@Entity
public class Member {
    @Id @GeneratedValue private Long id;
    private String name;
    public Member(String name) { this.name = name; }   // → 컴파일러가 기본 생성자 추가 안 함
}
```

- `findById` 시 Hibernate 는 리플렉션으로 **기본 생성자**를 호출해 빈 인스턴스를 만들고 필드를 채운다. 그런데 `Member(String)` 을 선언했으니 기본 생성자가 없다 → 컴파일은 되지만 **런타임 에러**.
- 해결: `protected Member() {}` 또는 `@NoArgsConstructor(access = AccessLevel.PROTECTED)`. protected 인 이유는 [[jpa-entity-default-constructor]].

## 오답·헷갈린 점

- **전**: "자바가 알아서" 만들어 준다 → **후**: **컴파일러(javac)** 가 컴파일 시점에. 모호한 주어는 면접에서 바로 파고들어진다.
- **전**: 기본 생성자가 `String name = new String();` 을 생성하고, int 는 "기본타입이라 생략된다" → **후**: `new String()` 이면 `""` 이라 `b.name.length()` 가 0 이어야 하지만 실제로는 NPE. `name` 은 `null`, `age` 는 `0` 이고, 이건 생성자가 아니라 **힙 할당 시점의 타입별 기본값**. int 도 생략이 아니라 `0` 이 들어간다.
- **전**: 기본 생성자 모양을 쓰라는 질문에 `String name = null; int age = 0;` (필드 선언) → **후**: `public Dog() { super(); }`. 힌트(Object 상속, 부모 생성자 먼저)를 두 번 받고도 `super()` 를 떠올리지 못함.
- **전**: (생성자 `Dog(String)` 만 있을 때) ① `new Dog("초코")` 안 된다 / ② `new Dog()` 된다, "컴파일러는 **매개변수 값이 없는 경우에만** 기본 생성자를 만든다" → **후**: 정반대. 컴파일러는 **호출부의 인자 유무가 아니라 클래스 선언부**를 본다. ① 된다, ② 컴파일 에러. 두 번 연속 같은 답 → 설명 후 "생성자가 하나도 없을 때만" 으로 스스로 정리.
- **전**: ②를 되게 하려면 `@Override` → **후**: 생성자 추가는 **오버로딩**. 생성자는 상속되지 않아 오버라이딩 불가. ⚠️ 10/09 [[overriding-vs-overloading-binding]] 에서 다룬 구분이 생성자 맥락에서 **재발**.
- **전**: 그다음 `Dog dog = new Dog();` → **후**: 그건 **호출**. 필요한 건 클래스 안의 **선언** `Dog() {}`. 선언/호출 혼동이 위 ①② 오답의 뿌리.
- **전**: JPA 엔티티에 기본 생성자 추가 = `@Getter` → **후**: `@Getter` 는 getter 생성. 필요한 건 `@NoArgsConstructor(access = AccessLevel.PROTECTED)`. ⚠️ 10/18 [[jpa-entity-default-constructor]] 에서 정리했던 내용이 **재발** — 이름으로는 아는데 "왜 이 어노테이션이 필요한가" 가 오늘 규칙과 연결돼 있지 않았다.
- 👍 "JPA 는 기본 생성자로 만들려고 한다" — 정답.

## Q&A

**Q. 기본 생성자는 누가, 언제 만드나?**
A. 컴파일러(javac)가 컴파일할 때, 클래스에 생성자가 하나도 선언돼 있지 않을 때만 만든다.

**Q. 기본 생성자의 모양은?**
A. `public Dog() { super(); }` — 매개변수 없음, 접근제어자는 클래스를 따름, 본문은 부모 생성자 호출 한 줄.

**Q. 생성자를 안 쓴 클래스에서 `new Dog()` 후 `b.name`, `b.age` 는?**
A. `null`, `0`. 생성자가 아니라 힙 할당 시점의 타입별 기본값이다.

**Q. `Dog(String)` 만 있을 때 `new Dog()` 는?**
A. 컴파일 에러. 생성자가 이미 있으니 컴파일러가 기본 생성자를 추가하지 않는다. `Dog() {}` 를 직접 선언(오버로딩)해야 한다.

**Q. 인자 있는 생성자만 있는 JPA 엔티티를 조회하면?**
A. 런타임 에러. Hibernate 가 리플렉션으로 기본 생성자를 호출하는데 없다. `@NoArgsConstructor(access = AccessLevel.PROTECTED)` 로 해결.

## 다음 학습

- `this(...)` 로 생성자 체이닝 — `Dog() { this("이름없음"); }` 와 `Dog() {}` 의 차이
- 부모에 기본 생성자가 없을 때 자식 생성자에서 생기는 컴파일 에러 (`super()` 자동 삽입과 연결)
- 필드 초기화 순서: 기본값 → 명시적 초기화(`int age = 1;`) → 초기화 블록 → 생성자 본문 → [[variable-kinds-initialization]]
- 선언 vs 호출, 오버로딩 vs 오버라이딩 다시 인출해 보기 → study-review
