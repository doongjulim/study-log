---
type: refined
slug: jpa-entity-default-constructor
tags: [기본생성자, 기본 생성자, default-constructor, no-arg-constructor, NoArgsConstructor, AccessLevel.PROTECTED, protected, 접근제어자, access-modifier, 리플렉션, reflection, newInstance, getDeclaredConstructor, Field.set, 엔티티생성, entity-instantiation, 프록시, proxy, 상속, inheritance, super(), 불변식, invariant, 생성자우회, final클래스, final-class, final메서드, final-method, 조용한null, record엔티티, Kotlin-allopen, no-arg-plugin, JPA명세, JPA, Hibernate, 면접답변]
topic: JPA 는 리플렉션으로 기본 생성자를 호출해 엔티티를 만들기 때문에 기본 생성자가 필요하고, 프록시(상속)는 허용하되 외부 new 는 막으려고 protected 로 둔다. 같은 뿌리에서 final 클래스·final 메서드·record 금지가 나온다
summary: Hibernate 는 엔티티 클래스를 모르는 라이브러리라 new 를 쓸 수 없고 인자 있는 생성자의 파라미터와 컬럼을 짝지을 수 없어서, 리플렉션으로 기본 생성자를 호출해 빈 객체를 만든 뒤 매핑 정보로 private 필드까지 리플렉션으로 채운다. 지연 로딩 프록시는 엔티티를 상속하므로 생성자 첫 줄 super() 가 부모 기본 생성자에 접근할 수 있어야 해 private 은 안 되고, public 이면 검증 생성자를 우회해 불변식이 깨지므로 protected 가 가장 좁은 정답이다(JPA 명세도 public 또는 protected). 같은 이유로 엔티티 클래스가 final 이면 프록시를 못 만들고, 메서드가 final 이면 가로채지 못해 프록시 자신의 null 필드를 조용히 돌려주며, record 는 final 클래스·기본 생성자 없음·필드 final 이라 엔티티가 될 수 없다.
contributors: [dongju]
source_refs:
  - https://jakarta.ee/specifications/persistence/3.1/
  - https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#entity-pojo
updated: 2026-10-18
---

# 엔티티의 기본 생성자와 protected

## 개념 정의

```java
@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Post {
    public Post(String title, String content, Member author, PostCategory category) { ... }   // 검증하는 생성자
}
```

**JPA 가 엔티티를 만드는 두 단계**

```java
Post post = Post.class.getDeclaredConstructor().newInstance();   // ① 리플렉션으로 기본 생성자 → 빈 객체
// ② 매핑 정보(@Column, 이름 규칙)로 컬럼↔필드를 알고 Field.set(post, 값) 으로 private 필드까지 채움
```

- 인자 있는 생성자를 쓸 수 없는 이유: 파라미터 이름은 컴파일 후 보통 사라지고, 생성자가 여러 개면 고를 기준이 없다.
- Hibernate 는 애플리케이션보다 먼저 만들어진 라이브러리라 코드에 `new Post()` 를 쓸 수 없다 → **리플렉션**.
- setter 가 없어도 되는 이유도 ② (필드에 직접).

## 작동 원리 — 왜 protected 인가

| 접근 제어자 | 결과 |
|---|---|
| `public` | 외부에서 `new Post()` → 검증 생성자를 우회하는 **뒷문** → 제목·작성자 없는 엔티티 → **불변식** 깨짐 |
| `protected` ✅ | 외부 `new` 는 막고(다른 패키지에선 컴파일 에러), **자식(프록시)** 의 `super()` 는 허용하는 가장 좁은 범위 |
| `private` | 지연 로딩 프록시는 엔티티를 **상속**한 클래스 → 생성자 첫 줄 `super()` 로 부모 기본 생성자를 불러야 하는데 접근 불가 → **프록시 생성 불가** |

- 리플렉션은 `setAccessible(true)` 로 private 생성자도 부를 수 있어 ①만 보면 private 도 될 것 같지만, 프록시(상속) 때문에 안 된다 ([[lazy-loading-proxy]]).
- JPA 명세도 기본 생성자를 `public` 또는 `protected` 로 요구한다.
- `@Embeddable` 값 타입도 JPA 가 같은 방식으로 만들므로 기본 생성자가 필요하다 ([[jpa-embedded-value-type]]).

## 트레이드오프 / 한계 — 프록시 = 상속에서 나오는 금지 사항

| 하면 안 되는 것 | 결과 |
|---|---|
| 엔티티 클래스 `final` | 상속 불가 → 프록시 생성 불가 → 지연 로딩 불가 |
| 엔티티 메서드 `final` | 프록시가 가로채지 못함 → 초기화 없이 부모 코드가 **프록시 자신의 필드**를 읽음 → **조용히 null** |
| `record` 를 엔티티로 | `final` 클래스 + 기본 생성자 없음·필드 `final` → 프록시도, JPA 의 생성·변경 방식도 불가 → DTO 로 사용 ([[java-record]], [[jpa-dto-projection]]) |

```
proxy.getNickname()        // final 메서드
  → 가로채지 못함 → target 초기화도 일어나지 않음
  → 부모 코드 실행: return this.nickname  (this = 프록시)
  → 프록시 자신의 필드는 null → null
```

- Kotlin 클래스·메서드는 기본이 `final` 이라 엔티티에 `allopen`/`no-arg` 컴파일러 플러그인이 필요한 것도 같은 이유.

> **면접 답변** — "JPA는 DB에서 조회한 행으로 엔티티를 만들 때, 어떤 생성자에 어떤 컬럼을 넣을지 알 수 없어서 리플렉션으로 기본 생성자를 호출해 빈 객체를 만들고, 매핑 정보를 보고 필드를 리플렉션으로 채웁니다. 그래서 기본 생성자가 반드시 필요합니다. private으로 두면 안 되는 이유는 지연 로딩 프록시가 엔티티를 상속한 클래스라서 생성자 첫 줄에서 super()를 호출해야 하기 때문이고, public이면 어디서나 new로 제목도 작성자도 없는 엔티티를 만들 수 있어 생성자가 지켜 주던 불변식이 깨집니다. 그래서 프록시에게는 열고 외부의 new는 막는 가장 좁은 범위인 protected로 둡니다."

> [!WARNING]
> **오답 코너**
> - **"JPA 는 new 로 빈 객체를 만든다"** — **리플렉션**으로 기본 생성자 호출.
> - **"필드는 같은 변수명을 가진 값으로 채운다"** — **매핑 정보**로 컬럼↔필드를 알고 리플렉션으로 채운다.
> - **"public 이면 불변성이 깨진다"** — **불변식**(규칙)이 깨진다. 불변성(값이 안 바뀜)과 다른 개념.
> - **"자식 생성자는 부모의 클래스를 호출 / private 이면 생성자를 만들 수 없다"** — 부모의 **생성자**(`super()`)를 호출하고, private 이면 **프록시를** 만들 수 없다.
> - **"final 메서드를 프록시에서 호출하면 부모의 값이 나온다"** — 부모의 **코드**가 **프록시 자신의 필드**(null)를 읽어 null.
> - **"record 가 엔티티가 안 되는 이유는 final 하나"** — + 기본 생성자 없음·필드 final.
> - 답할 때 "상속받아 사용", "아무 데서나 생성" 같은 현상만 말하지 말고 **프록시, super(), 불변식** 같은 원인의 이름으로.

## 복습 체크

- [ ] JPA 가 엔티티를 만드는 두 단계를 말할 수 있는가? 인자 있는 생성자를 못 쓰는 이유는?
- [ ] 기본 생성자를 public / protected / private 으로 두면 각각 어떻게 되는가?
- [ ] private 이 안 되는 이유를 "프록시", "상속", "super()" 로 설명할 수 있는가?
- [ ] public 이 안 되는 이유를 "불변식" 으로 설명할 수 있는가?
- [ ] 엔티티 클래스·메서드를 final 로 하면 각각 어떻게 되는가? final 메서드가 null 을 돌려주는 흐름은?
- [ ] record 를 엔티티로 쓸 수 없는 이유 두 가지는?

## 관련

[[lazy-loading-proxy]] · [[jpa-embedded-value-type]] · [[jpa-entity-equality]] · [[java-record]] · [[jpa-dto-projection]]
