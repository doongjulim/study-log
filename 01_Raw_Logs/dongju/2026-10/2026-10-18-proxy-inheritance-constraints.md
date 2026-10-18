---
date: 2026-10-18
type: raw
author: dongju
tags: [프록시, proxy, HibernateProxy, 상속, inheritance, final 클래스, final class, final 메서드, final method, 메서드 오버라이드, override, 가로채기, interception, 프록시 필드 null, 지연 로딩, lazy loading, record, Java record, 엔티티 제약, entity constraints, 기본 생성자, DTO, ByteBuddy]
topic: 지연 로딩 프록시가 엔티티를 "상속"해 메서드를 "가로채는" 방식에서 나오는 금지 사항 — final 클래스(프록시 불가), final 메서드(조용히 null), record(엔티티 불가)
summary: 프록시는 엔티티를 상속하고 메서드를 재정의해 가로챈다. 그래서 엔티티 클래스가 final 이면 상속할 수 없어 프록시를 만들 수 없고 지연 로딩이 불가하다. 메서드가 final 이면 프록시가 가로채지 못해 초기화 없이 부모 코드가 실행되고, 그 코드가 읽는 this 는 프록시 자신이라 필드가 null — 에러 없이 조용히 null 이 나온다. record 는 final 클래스이고 기본 생성자 없이 정규 생성자뿐이며 필드도 final 이라, 프록시도 JPA 의 생성(기본 생성자 + 필드 주입)·변경(변경 감지) 방식도 불가해 엔티티 대신 DTO 로 쓴다.
source: session
distilled: false
---

## 배운 개념

### 1. 뿌리 — 프록시 = 상속 + 메서드 가로채기

- 지연 로딩 프록시는 엔티티의 **자식 클래스**(`Member$HibernateProxy$...`). 메서드를 **재정의**해 가로채고 target 에 위임한다 (10/13).
- 기본 생성자를 `private` 으로 못 두는 이유와 같은 뿌리에서 아래 금지 사항이 나온다.

### 2. 금지 사항

| 하면 안 되는 것 | 결과 |
|---|---|
| 엔티티 클래스 `final` | 상속 불가 → **프록시 생성 불가** → 지연 로딩 불가 |
| 엔티티 메서드 `final` | 프록시가 가로채지 못함 → 초기화 없이 부모 코드가 **프록시 자신의 필드**(null)를 읽음 → **조용히 null** |
| `record` 를 엔티티로 | `final` 클래스 + 기본 생성자 없음·필드 `final` → 프록시도, JPA 의 생성·변경 방식도 불가 |

### 3. final 메서드가 null 을 돌려주는 흐름

```
proxy.getNickname()        // final 메서드
  → 프록시가 재정의할 수 없어 가로채지 못함 → 초기화(target 연결)도 일어나지 않음
  → Member 의 코드 실행: return this.nickname  (this = 프록시)
  → 프록시 자신의 필드는 끝까지 null → null
```

- 에러 없이 조용히 null — 10/13 `equals` 에서 `other.id`(필드 직접 접근)가 null 이던 것과 같은 원리.

### 4. record 가 엔티티가 될 수 없는 이유 두 가지

| record 의 특징 | 충돌하는 규칙 |
|---|---|
| 클래스가 `final` | 상속 불가 → 프록시 불가 |
| 정규 생성자(모든 필드)뿐, 기본 생성자 없음 + 필드 `final` | "기본 생성자로 빈 객체 → 필드 채우기" 생성 방식과 맞지 않음. 변경 감지로 값을 바꾸는 것도 불가 |

→ record 는 엔티티 대신 **DTO**(10/14 `PostSummary` 프로젝션)로 쓴다.

## 오답·헷갈린 점

- ✅ final 클래스 → 프록시 불가는 정확히 답함.
- **전**: final 메서드를 프록시에서 호출하면 "부모의 값이 나온다" → **후**: 부모의 **코드**가 실행되고, 그 코드가 읽는 필드는 **프록시 자신의 필드**라 **null**. 초기화도 안 일어남.
- **전**: record 가 엔티티가 안 되는 이유 = "final 값으로 사용되는 점" (1개) → **후**: ① final 클래스(프록시 불가) ② 기본 생성자 없음 + 필드 final(JPA 생성·변경 방식과 충돌).

## Q&A

- **Q. 엔티티 클래스를 final 로 하면?** A. 프록시를 못 만들어 지연 로딩 불가.
- **Q. 엔티티 메서드를 final 로 하면?** A. 가로채지 못해 프록시 자신의 null 필드를 읽음 → 조용히 null.
- **Q. record 를 엔티티로 못 쓰는 이유 두 가지?** A. final 클래스, 기본 생성자 없음·필드 final.

## 다음 학습

- Hibernate 가 런타임에 프록시를 만드는 방식(ByteBuddy)과 final 이 있을 때 남기는 경고 로그 확인
- Kotlin 엔티티에 `allopen` / `no-arg` 플러그인이 필요한 이유 (같은 원리)
- 연결: [[lazy-loading-proxy]], [[jpa-entity-equality]], [[java-record]], [[jpa-dto-projection]], [[2026-10-18-entity-default-constructor-protected]]
