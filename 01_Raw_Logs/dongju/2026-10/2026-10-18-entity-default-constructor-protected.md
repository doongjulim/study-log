---
date: 2026-10-18
type: raw
author: dongju
tags: [기본 생성자, default constructor, no-arg constructor, NoArgsConstructor, AccessLevel.PROTECTED, protected, 접근 제어자, access modifier, 리플렉션, reflection, newInstance, getDeclaredConstructor, Field.set, 엔티티 생성, entity instantiation, 매핑 정보, 프록시, proxy, 상속, inheritance, super(), 불변식, invariant, 생성자 우회, JPA 명세, study-board, 면접 답변]
topic: JPA 는 리플렉션으로 기본 생성자를 호출해 빈 엔티티를 만들고 필드를 채우므로 기본 생성자가 필요하다 — private 이면 프록시(상속)가 super() 를 못 부르고, public 이면 검증 생성자를 우회해 불변식이 깨지므로 protected
summary: Hibernate 는 엔티티 클래스를 모르는 라이브러리라 new 를 쓸 수 없고, 인자 있는 생성자는 파라미터와 컬럼을 짝지을 수 없어서, 리플렉션으로 기본 생성자를 호출해 빈 객체를 만든 뒤 매핑 정보를 보고 private 필드까지 리플렉션으로 채운다. 지연 로딩 프록시는 엔티티를 상속하므로 생성자 첫 줄의 super() 가 부모 기본 생성자에 접근할 수 있어야 해 private 은 안 되고, public 이면 서비스 코드에서 new Post() 로 제목·작성자 없는 엔티티를 만들어 불변식이 깨진다. 그래서 자식(프록시)에게만 열고 외부 new 는 막는 가장 좁은 범위인 protected 를 쓴다(JPA 명세도 public 또는 protected 요구). study-board 는 엔티티 17개와 값 타입 2개 모두 @NoArgsConstructor(access = PROTECTED).
source: session
distilled: true
distilled_at: 2026-10-18
---

## 배운 개념

### 1. study-board

```java
@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Post {
    public Post(String title, String content, Member author, PostCategory category) { ... }
}
```

- 엔티티 17개 + 값 타입 2개(`ProfileImage`, `NotificationPreference`) = 19곳 모두 동일.

### 2. 왜 기본 생성자가 필요한가 — JPA 가 엔티티를 만드는 두 단계

```java
// Hibernate 가 하는 일 (개념적으로)
Constructor<Post> c = Post.class.getDeclaredConstructor();   // ① 인자 없는 생성자를 찾아
Post post = c.newInstance();                                 //    빈 객체 생성 (리플렉션)
// ② 매핑 정보(@Column, 이름 규칙)로 컬럼↔필드를 알고 Field.set(post, 값) 으로 private 필드까지 채움
```

- 인자 있는 생성자를 못 쓰는 이유: 파라미터 이름은 컴파일 후 보통 사라지고(`arg0`…), 생성자가 여러 개면 고를 기준이 없다 → 어느 파라미터에 어느 컬럼을 넣을지 알 수 없다.
- Hibernate 는 study-board 보다 먼저 만들어진 라이브러리라 코드에 `new Post()` 를 쓸 수 없다 → **리플렉션**.
- setter 가 없어도 되는 이유도 ② (필드에 직접).

### 3. 왜 protected 인가

| 접근 제어자 | 결과 |
|---|---|
| `public` | 외부에서 `new Post()` 가능 → 검증하는 생성자를 우회하는 **뒷문** → 제목·작성자 없는 글 → **불변식** 깨짐 |
| `protected` ✅ | 외부 `new` 는 막고(다른 패키지에선 컴파일 에러), **자식(프록시)** 의 `super()` 는 허용하는 가장 좁은 범위 |
| `private` | 지연 로딩 프록시는 엔티티를 **상속**한 클래스 → 생성자 첫 줄에서 `super()` 로 부모 기본 생성자를 호출해야 하는데 접근 불가 → **프록시 생성 불가** |

- JPA 명세도 기본 생성자를 `public` 또는 `protected` 로 요구한다.
- 리플렉션은 `setAccessible(true)` 로 private 생성자도 호출할 수 있으니 ①만 보면 private 도 될 것 같지만, 프록시(상속) 때문에 안 된다.

### 4. 면접 답변 (30초)

> "JPA는 DB에서 조회한 행으로 엔티티를 만들 때, 어떤 생성자에 어떤 컬럼을 넣을지 알 수 없어서 리플렉션으로 기본 생성자를 호출해 빈 객체를 만들고, 매핑 정보를 보고 필드를 리플렉션으로 채웁니다. 그래서 기본 생성자가 반드시 필요합니다.
> private으로 두면 안 되는 이유는 지연 로딩 프록시가 엔티티를 상속한 클래스라서, 생성자 첫 줄에서 부모의 기본 생성자를 super()로 호출해야 하기 때문입니다. 반대로 public이면 서비스 코드 어디서나 new Post()로 제목도 작성자도 없는 엔티티를 만들 수 있어, 생성자가 지켜 주던 불변식이 깨집니다.
> 그래서 프록시에게는 열고 외부의 new는 막는 가장 좁은 범위인 protected로 두었고, study-board의 엔티티와 값 타입 19곳 모두 @NoArgsConstructor(access = PROTECTED)로 통일했습니다."

## 오답·헷갈린 점

- **전**: JPA 는 "new 로" 빈 객체를 만든다 → **후**: **리플렉션**으로 기본 생성자 호출.
- **전**: 필드는 "동일한 변수명을 가진 값으로" 채운다 → **후**: **매핑 정보**로 컬럼↔필드를 알고 리플렉션 `Field.set`.
- **전**: public 이면 "**불변성**이 깨진다" → **후**: **불변식**(규칙)이 깨진다. ⚠️ **같은 날 @Embedded 인터뷰에 이어 2회째** 혼동.

| | 뜻 | 이 주제에서 |
|---|---|---|
| 불변식 (invariant) | 객체가 항상 지켜야 하는 규칙 | 검증 생성자를 우회하면 깨짐 ← 정답 |
| 불변성 (immutability) | 만든 뒤 값이 안 바뀜 | 무관 |

- **전**: 자식 생성자는 첫 줄에서 "부모의 **클래스**를 호출" → **후**: 부모의 **생성자**(`super()`).
- **전**: private 이면 "**생성자를** 만들 수 없다" → **후**: **프록시를** 만들 수 없다.
- **전**: 최종 답변에서 "프록시", "super()", "불변식" 을 쓰지 않고 "상속받아서 사용", "아예 상관없는 곳에서 생성" 처럼 현상만 말함 → **후**: 면접에서 기대하는 건 **원인의 이름**. 핵심 단어를 넣어 말할 것.
- ⚠️ 반복 테마: 현상은 맞게 짚는데 **용어로 환원하지 못함** (10/16 "다 지워진다", 10/17 "테이블"에 이어).

## Q&A

- **Q. Hibernate 가 인자 있는 생성자를 못 쓰는 이유?** A. 파라미터와 컬럼을 짝지을 수 없음.
- **Q. JPA 가 엔티티를 만드는 순서?** A. 리플렉션으로 기본 생성자 → 매핑 정보로 필드를 리플렉션으로 채움.
- **Q. public 이면?** A. 검증 생성자 우회 → 불변식 깨짐.
- **Q. private 이면?** A. 프록시(상속)의 super() 가 접근 불가 → 프록시 생성 불가.

## 다음 학습

- 생성자 대신 **정적 팩토리 메서드**(`Comment.forPost(...)`, `Retrospective.write(...)`)로 엔티티를 만드는 이유
- 값 타입(`@Embeddable`)에도 기본 생성자가 필요한 이유는 엔티티와 같은가
- 연결: [[lazy-loading-proxy]], [[2026-10-18-embedded-value-type-why]], [[2026-10-18-proxy-inheritance-constraints]]
