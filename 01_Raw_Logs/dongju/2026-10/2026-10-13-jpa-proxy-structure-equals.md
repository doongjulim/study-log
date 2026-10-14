---
date: 2026-10-13
type: raw
author: dongju
tags: [프록시, proxy, HibernateProxy, 지연 로딩, LAZY, 프록시 초기화, proxy initialization, target, 위임, delegation, 상속, inheritance, 자식 클래스, subclass, LazyInitializer, getClass, instanceof, equals, 필드 직접 접근, field access, getter, 패턴 매칭, pattern matching, 연산자 우선순위, operator precedence, 엔티티 동등성, entity equality]
topic: 지연 로딩 프록시의 정체(상속), 초기화 시점, target 위임 구조 — 그리고 그 구조 때문에 equals 에서 getClass·필드 직접 접근이 깨지는 이유
summary: 프록시는 Hibernate 가 런타임에 만든 엔티티의 자식 클래스로, 처음엔 FK 에서 얻은 id 만 알고 target 은 null 이다. ID 외 값을 처음 읽을 때 영속성 컨텍스트가 DB 를 조회해 target 을 채우고 이후 호출은 target 에 위임하며, 프록시 자신의 필드는 끝까지 null 이다. 프록시는 메서드만 가로챌 수 있으므로 equals 는 getClass 대신 instanceof, 필드 대신 getter 로 비교해야 한다.
source: session
distilled: false
---

## 배운 개념

### 1. LAZY 조회 직후 `post.getMember()` 가 반환하는 것

- `null` 이 아니라 **프록시 객체** — `getClass()` 가 `Member$HibernateProxy$...` 로 찍힘.
- 프록시는 Hibernate 가 런타임에 만든 **`Member` 의 자식 클래스**(상속) → 그래서 `private Member member` 필드에 그대로 대입된다.
- 처음 아는 건 **id 뿐**. `select * from post` 결과의 FK `member_id` 에서 얻은 값.

### 2. 초기화 시점

```java
postRepository.findAll();   // ① 쿼리 O
post.getTitle();            // ② 쿼리 X (Post 는 이미 조회됨)
post.getMember();           // ③ 쿼리 X (프록시를 꺼내기만 함)
    .getName();             // ④ 쿼리 O (ID 외의 값을 처음 읽음 → 초기화)
```

### 3. 구조 — target 참조와 위임

```
 post.member
     │
     ▼
┌───────────────────────┐            ┌────────────────────┐
│ Member$HibernateProxy │   target   │ Member (진짜 엔티티) │
│  id 필드   = null      │ ─────────▶ │  id = 1            │
│  name 필드 = null      │            │  name = "동주"      │
└───────────────────────┘            └────────────────────┘
       ▲ getName() 호출                  ▲ 초기화 시점에 DB 조회해서 생성
```

1. 프록시는 진짜 엔티티에 대한 **참조**를 `target` 으로 가진다. 처음엔 `target == null`.
2. `getName()` 호출 → 영속성 컨텍스트에 **초기화** 요청 → DB 조회로 진짜 `Member` 생성 → `target` 에 연결.
3. 이후 호출은 자기 필드가 아니라 **target 에 위임**해서 결과를 돌려준다.
4. 그래서 프록시 자신의 필드는 **끝까지 null** — 리플렉션으로 프록시의 `name` 필드를 직접 보면 초기화 후에도 null.
5. 프록시가 "알고 있는" id 는 자기 `id` 필드가 아니라 내부 초기화기(LazyInitializer)에 있고, `getId()` 를 가로채서 돌려준다.

### 4. 핵심 원리: 프록시는 메서드만 가로챌 수 있다

- 상속으로 자식이 재정의(override)할 수 있는 건 **메서드**뿐, **필드 접근은 불가**.

```java
other.getId()   // 메서드 → 프록시가 가로챔 → 1
other.id        // 필드   → 가로챌 수 없음 → 프록시 자신의 필드 → null
```

### 5. 이 구조가 equals 를 깨뜨린다

```java
// ❌ 버그 버전
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;   // (A)
    Member other = (Member) o;
    return Objects.equals(this.id, other.id);                     // (B)
}
```

- **(A)**: `real.getClass()` = `Member.class`, `proxy.getClass()` = `Member$HibernateProxy$xxx.class` → 부모·자식은 서로 다른 Class 객체 → `false`.
- **(B)**: (A) 를 고쳐도 `other.id` 는 프록시 자신의 필드 → `null` → `false`.

```java
// ✅ 수정 버전 (Java 17 패턴 매칭)
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Member other)) return false;      // 상속까지 허용 + null 체크 겸용
    return Objects.equals(this.getId(), other.getId());  // getter → 프록시가 가로챔
}
```

- `getClass()` 비교 = "정확히 같은 클래스인가?" / `instanceof` = "그 클래스이거나 **자식**인가?"
- `instanceof` 는 `o == null` 이면 자동 `false` → null 체크 생략 가능.

## 오답·헷갈린 점

- **전**: `getMember()` 반환값 = "DB 에 ID 로 접근해 얻은 결과, member 엔티티가 들어있음" → **후**: `getMember()` 는 **프록시만 반환**하고 쿼리 X. DB 조회는 **ID 외의 값을 읽을 때**(초기화). 꺼내기만 해도 조회하면 지연 로딩의 의미가 없음.
- **전**: 프록시와 `Member` 의 관계 = "조인" → **후**: 조인은 DB 테이블 개념. 자바 타입 관계는 **상속**(Dog → Animal 과 같은 이유로 대입 가능).
- **전**: "프록시는 target 정보만 위임받아 있다"(정보를 받아서 들고 있다) → **후**: 방향 반대. 프록시는 target **참조**만 들고, 호출을 target 에 **넘긴다(위임)**.
- **전**: `other.id` 는 "프록시가 이미 가지고 있는 값이라 같다고 리턴될 수 있다" → **후**: 직전에 스스로 말한 "프록시 필드는 끝까지 null" 과 모순. 필드 접근은 가로챌 수 없어서 **null** → equals `false`.
- **전**: (A) 수정안 = "`getClassEhter()`" (존재하지 않는 메서드) → **후**: 상속 관계 판단 연산자는 `instanceof`.
- **전**: `if (o == null || getClass() != o.getClass() || o instanceof Member) return false;` → **후**: 기존 getClass 조건을 남기고 조건까지 뒤집혀서 **모든 Member 에 false**. getClass 조건 삭제 + 부정 필요.
- **전**: `if (!o instanceof Member)` → **후**: 연산자 우선순위로 `!o` 가 먼저 계산 → 컴파일 에러. **`!(o instanceof Member)`** 괄호 필수.
- ⚠️ **재발 주의**: 2026-09-10 세션(equals-hashCode / JPA 엔티티 equals)에서 이미 "getClass 대신 instanceof" 를 다뤘는데 이번에 떠올리지 못함. **"왜 instanceof 인가 → 프록시는 자식 클래스라서"** 인과 고리로 묶어 기억할 것.
- ⚠️ 반복 테마: 객체 레벨(상속·위임) 개념을 DB 레벨(조인) 용어로 설명 — 지난 세션들의 "JVM 레벨 vs DB 레벨 혼동"과 같은 계열.

## Q&A

- **Q. LAZY 조회 직후 `post.getMember()` 는 무엇을 반환?** A. `Member` 를 상속한 프록시. id 만 알고 나머지는 비어 있음.
- **Q. 실제 DB 조회는 언제?** A. ID 외의 값을 처음 읽을 때(초기화).
- **Q. 프록시가 Member 타입 필드에 들어갈 수 있는 이유?** A. 상속(자식 클래스).
- **Q. 초기화 후 `getName()` 의 값은 어디서?** A. 프록시가 target(진짜 엔티티)에 위임해서 가져옴. 프록시 자신의 name 필드는 null.
- **Q. 같은 id 인데 `real.equals(proxy)` 가 false 인 이유?** A. (A) getClass 가 부모/자식으로 다름, (B) 필드 직접 접근은 프록시가 못 가로채서 null.
- **Q. 어떻게 고치나?** A. `!(o instanceof Member other)` + `getId()` 로 비교.

## 다음 학습

- 리플렉션 예제 직접 실행해서 프록시 필드가 null 인지 확인
- `Hibernate.getClass()`, `Hibernate.unproxy()` 의 용도
- 같은 영속성 컨텍스트에서 `em.getReference()` 후 `em.find()` 하면 프록시가 반환되는 이유(동일성 보장)
- 연결: [[lazy-loading-proxy]], [[jpa-entity-equality]], [[equals-hashcode-contract]], [[2026-10-13-why-lazy-default-fetch-join]]
