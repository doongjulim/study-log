---
type: refined
slug: enum-constant-specific-polymorphism
tags: [enum, 이넘, 열거형, 상수별메서드구현, constant-specific-method, constant-specific-class-body, 다형성, polymorphism, 익명클래스, anonymous-class, 익명자식클래스, abstract-method, 추상메서드, getClass, getDeclaringClass, SearchType$1, 동적바인딩, dynamic-binding, switch, switch식, switch-expression, default, 컴파일에러, OCP, 개방폐쇄원칙, SOLID, 전략패턴, strategy-pattern, 스프링빈, spring-bean, Autowired, 클래스초기화, class-initialization, static싱글턴, 면접답변]
topic: enum 상수마다 다른 동작을 구현하면 왜 다형성인가 — 상수는 enum 을 상속한 익명 자식 클래스의 객체이고, 구현 누락이 컴파일 에러가 되지만 스프링 빈은 주입할 수 없다
summary: 몸통을 가진 enum 상수는 컴파일 시 enum 을 상속한 익명 자식 클래스(SearchType$1)가 되어 추상 메서드를 오버라이드하므로, searchType.matches() 한 줄은 일반 오버라이딩과 똑같이 동적 바인딩된다. switch(default) 대비 새 상수의 구현 누락이 컴파일 에러로 막히고 호출부가 안 바뀌어 OCP 를 만족한다. 다만 상수 객체는 JVM 이 클래스 초기화 때 만드는 static 싱글턴이라 스프링 빈을 주입할 수 없으므로, 의존성이 필요한 동작은 전략 패턴 빈으로 옮긴다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-8.html#jls-8.9.1
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Enum.html#getDeclaringClass()
updated: 2026-10-09
---

# enum 상수별 구현과 다형성

## 개념 정의

```java
public enum SearchType {
    TITLE {
        @Override
        public boolean matches(Post post, String keyword) {
            return post.getTitle().contains(keyword);
        }
    },
    WRITER {
        @Override
        public boolean matches(Post post, String keyword) {
            return post.getWriter().equals(keyword);
        }
    };

    public abstract boolean matches(Post post, String keyword);
}

boolean hit = searchType.matches(post, keyword);   // 호출 코드는 한 줄
```

**하나의 호출 코드**(`searchType.matches`) → **실제 상수 객체**(TITLE/WRITER) → **여러 구현**(제목/작성자 비교). `List.add` 가 ArrayList/LinkedList 에 따라 달라지는 것과 같은 구조다 ([[overriding-vs-overloading-binding]]).

## 작동 원리 — enum 안에 숨은 부모-자식

- `abstract` 메서드가 있는 클래스는 직접 `new` 할 수 없다 → `TITLE` 은 `SearchType` 그 자체의 객체일 수 없다.
- `TITLE { ... }` 는 `new Comparator<>() { ... }` 와 같은 **익명 클래스** 문법이고, `@Override` 가 부모-자식 관계의 증거다.

컴파일러가 풀어 쓰면 대략:

```java
abstract class SearchType {
    public abstract boolean matches(Post p, String kw);
}
final class SearchType$1 extends SearchType {   // TITLE 의 진짜 클래스
    @Override public boolean matches(Post p, String kw) { return p.getTitle().contains(kw); }
}
final class SearchType$2 extends SearchType {   // WRITER 의 진짜 클래스
    @Override public boolean matches(Post p, String kw) { return p.getWriter().equals(kw); }
}
static final SearchType TITLE  = new SearchType$1();
static final SearchType WRITER = new SearchType$2();
```

```java
SearchType.TITLE.getClass()                      // class SearchType$1   ← 자식 클래스
SearchType.TITLE.getClass() == SearchType.class  // false
SearchType.TITLE.getDeclaringClass()             // class SearchType     ← 소속 enum 은 이걸로
SearchType.TITLE instanceof SearchType           // true
```

→ "enum 에 메서드를 넣은 것" 이 아니라 진짜 **서브타입 다형성**. 호출은 `invokevirtual` → 실제 클래스의 vtable → **동적 바인딩**.

## 트레이드오프 / 한계

### switch 대비 — 컴파일러를 체크리스트로

```java
public boolean matches(SearchType type, Post post, String kw) {
    switch (type) {
        case TITLE:  return post.getTitle().contains(kw);
        case WRITER: return post.getWriter().equals(kw);
        default:     return false;
    }
}
```

`CONTENT` 상수 이름만 추가하고 구현을 깜빡하면:

| | switch 문 (`default` 있음) | 상수별 구현 |
|---|---|---|
| 결과 | 에러 없이 `default` → 항상 `false` → 사용자 신고 전까지 모름 | **컴파일 에러** |
| 수정 범위 | 검색·화면 라벨·쿼리 조립 등 흩어진 switch 전부 | 상수 한 곳 |
| 호출부 | 수정 필요 | 그대로 → **OCP** |

```
error: enum constant CONTENT must implement abstract method matches(Post,String)
```

- Java 14+ **switch 식**(`return switch (type) { case TITLE -> ...; };`)은 `default` 없이 쓰면 모든 상수를 다뤘는지 컴파일러가 검사한다. 그래도 분기가 여러 곳에 흩어지는 문제는 남는다.

### 스프링 빈을 주입할 수 없다

```java
TITLE {
    @Autowired PostRepository postRepository;   // ❌ 동작 안 함
}
```

- **컴파일러**는 `SearchType$1.class` 같은 클래스 파일까지만 만든다.
- **객체**는 실행 중 **JVM 이** `SearchType` 을 처음 쓸 때 클래스 초기화(static 초기화)로 만든다.
- 스프링 컨테이너가 만들지도 관리하지도 않으므로 주입 불가. JVM 전역 static 싱글턴이라 의존성을 두는 것 자체가 어색하다 (static 유틸에 주입이 안 되는 것과 같은 이유 — [[static-util-vs-spring-bean]]).

→ 의존성이 필요한 동작은 **전략 패턴 빈**으로:

```java
public interface SearchStrategy {
    SearchType type();
    List<Post> search(String keyword);
}

@Component
class TitleSearchStrategy implements SearchStrategy {
    private final PostRepository postRepository;   // ✅ 주입 가능
    public SearchType type() { return SearchType.TITLE; }
    public List<Post> search(String kw) { return postRepository.findByTitleContaining(kw); }
}

@Service
class SearchService {
    private final Map<SearchType, SearchStrategy> strategies;
    SearchService(List<SearchStrategy> list) {
        strategies = list.stream().collect(Collectors.toMap(SearchStrategy::type, s -> s));
    }
    public List<Post> search(SearchType type, String kw) {
        return strategies.get(type).search(kw);
    }
}
```

enum 은 **"무엇을"(타입 식별)**, 빈은 **"어떻게"(의존성이 필요한 동작)**.

### 면접 답변 뼈대

> "검색 타입마다 다른 비교 로직을 enum 상수에 직접 구현했습니다. 몸통을 가진 enum 상수는 컴파일하면 enum 을 상속한 익명 자식 클래스가 되고, 그 클래스가 추상 메서드를 오버라이드합니다. 그래서 `searchType.matches()` 한 줄이 실행 시점에 실제 상수의 클래스를 보고 구현을 고르는 **동적 바인딩**이 일어나고, 이게 다형성입니다.
> switch 대신 이렇게 한 이유는, 새 검색 타입을 추가할 때 구현을 빠뜨리면 **컴파일 에러**가 나고 사용하는 쪽 코드는 수정할 필요가 없기 때문입니다. 다만 enum 상수는 스프링 빈이 아니라서 Repository 가 필요한 로직은 전략 패턴 빈으로 분리합니다."

> [!WARNING]
> **오답 코너**
> - **"TITLE 의 `matches` 구현은 2개"** — TITLE 은 1개. 2개는 enum 전체 합계다.
> - **"`TITLE.getClass()` 는 `SearchType`"** — `SearchType` 은 abstract 라 직접 객체를 만들 수 없다. 실제는 **`SearchType$1`**. 소속 enum 은 `getDeclaringClass()`.
> - **"`TITLE { ... }` 는 추상형 인터페이스를 만든다"** — **SearchType 을 상속한 익명 자식 클래스**를 만든다. `@Override` 가 증거.
> - **"enum 객체는 컴파일 시 컴파일러가 만든다"** — 컴파일러는 클래스 파일까지. 객체는 **실행 중 JVM 이 클래스 초기화 때** 만든다. 어느 쪽이든 스프링은 관여하지 않는다.

## 복습 체크

- [ ] 상수별 구현이 다형성인 이유를 "익명 자식 클래스 + 오버라이딩 + 동적 바인딩" 으로 설명할 수 있는가?
- [ ] `TITLE.getClass()`, `getDeclaringClass()`, `instanceof SearchType` 의 결과는?
- [ ] 새 상수의 구현을 빠뜨렸을 때 switch(default) 와 상수별 구현의 차이는?
- [ ] switch 식(Java 14+)은 이 문제를 어디까지 해결하는가?
- [ ] enum 상수에 `@Autowired` 가 안 되는 이유와 대안은?

## 관련

[[overriding-vs-overloading-binding]] · [[static-util-vs-spring-bean]] · [[stream-vs-for-loop]]
