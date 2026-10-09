---
date: 2026-10-09
type: raw
author: dongju
tags: [enum, 이넘, 열거형, 상수별 메서드 구현, constant-specific method, constant-specific class body, 다형성, polymorphism, 서브타입 다형성, subtype polymorphism, 익명 클래스, anonymous class, 익명 자식 클래스, abstract method, 추상 메서드, getClass, getDeclaringClass, SearchType$1, 동적 바인딩, dynamic binding, switch, default, 컴파일 에러, OCP, 개방 폐쇄 원칙, SOLID, 전략 패턴, strategy pattern, 스프링 빈, spring bean, Autowired, 클래스 초기화, static 싱글턴, 면접 답변]
topic: enum 상수마다 다른 동작을 구현하는 것이 왜 다형성인가 — 익명 자식 클래스 + 오버라이딩 + 동적 바인딩, switch 대비 이점과 스프링 빈 한계
summary: 다형성은 하나의 호출 코드가 점(.) 왼쪽의 실제 객체에 따라 여러 구현 중 하나로 실행되는 것이다. 몸통을 가진 enum 상수는 컴파일 시 enum 을 상속한 익명 자식 클래스(SearchType$1)가 되어 추상 메서드를 오버라이드하므로, searchType.matches() 는 일반 오버라이딩과 똑같이 동적 바인딩된다. switch 대비 새 상수의 구현 누락이 컴파일 에러로 막히고 호출부가 안 바뀌어 OCP 를 만족하지만, enum 상수는 JVM 이 만드는 static 싱글턴이라 스프링 빈 주입이 안 되므로 의존성이 필요한 동작은 전략 패턴 빈으로 옮긴다.
source: session
distilled: false
---

## 배운 개념

### 1. 다형성의 정의 — "하나 / 실제 / 여러"

> **하나의 호출 코드**가, 점(.) 왼쪽의 **실제 객체**에 따라, **여러 구현** 중 하나로 실행되는 것.

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

| 칸 | `List` 예시 | `SearchType` 예시 |
|---|---|---|
| **하나의 호출 코드** | `list.add(0, "X")` | `searchType.matches(post, kw)` |
| **실제 객체** | ArrayList / LinkedList | TITLE / WRITER |
| **여러 구현** | 같은 배열 안 shift / 새 Node 만들고 first 연결 | 제목 비교 / 작성자 비교 |

- 상수 하나는 구현을 **딱 1개**만 가진다. "여러 개"인 건 한 줄의 호출이 실행할 수 있는 로직이다.
- 호출하는 쪽은 상대가 누군지 **모른 채** 같은 한 줄을 쓴다.

```java
list . add ( 0, "X" )
 ↑      ↑      ↑
 ①받는 객체  ②메서드 이름  ③인자
```

결정권은 ① 점 **왼쪽의 받는 객체**에 있다. ③ 인자(`"X"`)는 ArrayList 든 LinkedList 든 똑같이 받으므로 구현을 정할 수 없다.

### 2. 메커니즘 — enum 안에 숨은 부모-자식 구조

- `abstract` 메서드가 있는 클래스는 `new` 로 직접 객체를 만들 수 없다 → `TITLE` 은 `SearchType` 그 자체의 객체일 수 없다.
- `TITLE { ... }` 블록 = `new Comparator<>() { ... }` 와 같은 **익명 클래스** 문법. `@Override` 가 부모-자식 관계의 증거.

컴파일러가 풀어 쓰면 대략:

```java
abstract class SearchType {                      // 부모 (abstract 메서드 보유)
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
SearchType.TITLE.getDeclaringClass()             // class SearchType     ← 소속 enum
SearchType.TITLE instanceof SearchType           // true
```

→ `ArrayList`/`LinkedList` 가 `List` 를 구현한 것과 **같은 구조**가 익명 클래스로 숨어 있다. "enum 에 메서드를 넣은 것" 이 아니라 진짜 **서브타입 다형성**.

### 3. 실행 — 동적 바인딩

```java
public List<Post> search(List<Post> posts, SearchType searchType, String keyword) {
    return posts.stream()
            .filter(post -> searchType.matches(post, keyword))
            .toList();
}
```

```
컴파일 시점:  "SearchType 타입에 matches 가 있나?" → 있음 → 통과 (어느 몸통인지는 모름)
            바이트코드: invokevirtual SearchType.matches
실행 시점:    searchType 이 가리키는 객체 → SearchType$1 → 그 클래스의 matches 실행 (vtable)
```

→ 오버로딩과의 구분은 [[overriding-vs-overloading-binding]] (raw: 2026-10-09-overriding-vs-overloading-binding).

### 4. switch 대비 — 컴파일러를 체크리스트로

```java
// (A) switch 방식 — 사용하는 쪽에서 분기
public boolean matches(SearchType type, Post post, String kw) {
    switch (type) {
        case TITLE:  return post.getTitle().contains(kw);
        case WRITER: return post.getWriter().equals(kw);
        default:     return false;
    }
}
```

팀원이 `CONTENT` 상수 이름만 추가하고 구현을 깜빡하면:

| | (A) switch (`default` 있음) | (B) 상수별 구현 |
|---|---|---|
| 결과 | 에러 없이 `default` → 항상 `false` → 사용자 신고 전까지 모름 | **컴파일 에러** |
| 수정 범위 | 검색·화면 라벨·쿼리 조립 등 흩어진 switch 전부 | 상수 한 곳만 |
| 호출부 | 수정 필요 | `searchType.matches(...)` 그대로 → **OCP** |

```
error: enum constant CONTENT must implement abstract method matches(Post,String)
```

- 참고: Java 14+ **switch 식**(`return switch (type) { case TITLE -> ...; };`)은 `default` 없이 쓰면 enum 의 모든 상수를 다뤘는지 컴파일러가 검사한다. 그래도 분기가 여러 곳에 흩어지는 문제는 남는다.

### 5. 한계 — 스프링 빈을 주입할 수 없다

```java
TITLE {
    @Autowired PostRepository postRepository;   // ❌ 동작 안 함
    ...
}
```

- **컴파일러**가 만드는 건 `SearchType$1.class` 같은 클래스 파일까지.
- **객체**는 **실행 중 JVM 이** `SearchType` 클래스를 처음 사용할 때 클래스 초기화(static 초기화)로 `new SearchType$1()` 해서 만든다.
- 스프링 컨테이너는 이 객체를 만들지도, 관리하지도 않는다 → `@Autowired` 불가. 게다가 JVM 전역 static 싱글턴이라 의존성을 넣는 것 자체가 어색하다.

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
        return strategies.get(type).search(kw);         // 다형성 그대로
    }
}
```

enum 은 **"무엇을 검색하나"(타입 식별)**, 빈은 **"어떻게 검색하나"(의존성이 필요한 동작)**.

### 6. 면접 답변 뼈대

> "검색 타입마다 다른 비교 로직을 enum 상수에 직접 구현했습니다. 몸통을 가진 enum 상수는 컴파일하면 enum 을 상속한 익명 자식 클래스가 되고, 그 클래스가 추상 메서드를 오버라이드합니다. 그래서 `searchType.matches()` 한 줄이 실행 시점에 실제 상수의 클래스를 보고 구현을 고르는 **동적 바인딩**이 일어나고, 이게 다형성입니다.
> switch 대신 이렇게 한 이유는, 새 검색 타입을 추가할 때 구현을 빠뜨리면 **컴파일 에러**가 나고 사용하는 쪽 코드는 수정할 필요가 없기 때문입니다. 다만 enum 상수는 스프링 빈이 아니라서 Repository 가 필요한 로직은 전략 패턴 빈으로 분리합니다."

(TODO: study-board 에서 실제로 쓴 enum 이름과 "switch 였다면 이런 버그" 장면 붙이기)

## 오답·헷갈린 점

- **전**: 다형성 = "**하나의 상수**로부터 여러 가지 메소드의 동작" → **후**: 상수 하나는 구현 **1개**. 하나인 것은 **호출 코드**, 여러 개인 것은 **구현**.
- **전**: `TITLE` 이 가진 `matches` 구현은 **2개** → **후**: 1개. 2개는 enum 전체 합계.
- **전**: "하나의 **상수**가 실제 들어온 **변수의 값**에 따라 여러 **메소드**로 실행" (재정의 시도에서도 '상수' 유지) → **후**: 하나의 **호출 코드**가 실제 **객체**에 따라 여러 **구현**으로. 힌트 2번 후에도 안 바뀌어 정답 제시.
- **전**: `list.add(0, "X")` 에서 실제 객체 = **`"X"`**, 여러 구현 = **람다식** → **후**: `"X"` 는 **인자**(점 오른쪽 괄호 안). 결정권은 점 **왼쪽** 받는 객체(ArrayList / LinkedList). 람다는 코드에 없다.
- **전**: `add(0, "X")` 의 구현 차이 = "ArrayList 에 담김" → **후**: ArrayList 는 같은 배열 안에서 뒤 원소 shift, LinkedList 는 새 Node + first 연결. (빈칸 힌트 후 정답 — 2026-10-07 노트 회상)
- **전**: `TITLE.getClass()` 는 **`SearchType`** → **후**: abstract 라 직접 객체 불가 → 실제는 **`SearchType$1`**. "abstract 는 new 불가" 를 스스로 맞혀 놓고 모순되는 답.
- **전**: `TITLE { ... }` 는 **추상형 인터페이스**를 만든다 → **후**: **SearchType 을 상속한 익명 자식 클래스**. `@Override` 가 증거.
- **전**: enum 객체는 **컴파일 시 컴파일러가** 만든다 → **후**: 컴파일러는 클래스 파일까지. 객체는 **실행 중 JVM 이 클래스 초기화 때** 만든다. 결론(스프링 빈 주입 불가)은 정답.
- **정답 맞힘**: abstract 클래스 new 불가 / 동적 바인딩은 컴파일 시 알 수 없고 호출 시점에 결정 / switch 누락 → 조용히 false, 상수별 구현 누락 → 컴파일 에러.

## Q&A

- **Q. enum 상수마다 다른 동작을 구현했는데, 이게 왜 다형성인가?**
  A. 몸통 있는 상수는 enum 을 상속한 익명 자식 클래스의 객체이고, 추상 메서드를 오버라이드한다. 호출은 실제 클래스를 보고 동적 바인딩된다 → 하나의 호출 코드, 여러 구현.
- **Q. `TITLE.getClass() == SearchType.class` 는?**
  A. false. `SearchType$1`. 소속 enum 은 `getDeclaringClass()`.
- **Q. 컴파일러는 `searchType.matches()` 가 어느 몸통을 부를지 아나?**
  A. 모른다. 실행 시점에 받는 객체의 실제 클래스로 결정(동적 바인딩).
- **Q. switch 대비 이점은?**
  A. 구현 누락이 컴파일 에러, 호출부 무수정(OCP), 분기 산재 없음.
- **Q. enum 상수에 `@Autowired` 를 쓸 수 있나?**
  A. 없다. JVM 이 클래스 초기화 때 만든 static 싱글턴이라 스프링이 관리하지 않는다 → 전략 패턴 빈.

## 다음 학습

- 상수별 구현 vs **생성자 + 람다 필드** (`TITLE((p, kw) -> ...)`) 비교
- `EnumMap` / `EnumSet` 이 빠른 이유 (ordinal 기반 배열)
- enum 싱글턴이 리플렉션·직렬화에 안전한 이유
- `invokevirtual` vs `invokeinterface` vs `invokestatic`, vtable 과 JIT 인라이닝
