---
type: refined
slug: jpa-id-generation-strategy
tags: [식별자생성전략, 식별자 생성 전략, id-generation-strategy, GeneratedValue, GenerationType, IDENTITY, SEQUENCE, TABLE, AUTO, auto_increment, identity-column, 시퀀스, SequenceGenerator, allocationSize, pooled-optimizer, nextval, 즉시INSERT, immediate-insert, 쓰기지연, write-behind, 1차캐시, first-level-cache, getGeneratedKeys, JDBC배치, JDBC-batch, hibernate.jdbc.batch_size, batch-insert, 네트워크왕복, round-trip, saveAll, 대량적재, bulk-insert, JdbcTemplate.batchUpdate, order_inserts, id구멍, id-gap, PostgreSQL, MySQL, JPA, Hibernate, 면접답변]
topic: IDENTITY 는 INSERT 를 해야 id 가 생기는데 1차 캐시는 id 를 키로 쓰므로 persist 순간 INSERT 가 즉시 나가 쓰기 지연·JDBC 배치가 안 된다 — SEQUENCE + allocationSize 와의 비교, 고르는 기준
summary: IDENTITY 는 DB 가 INSERT 시점에 id 를 만들고, SEQUENCE 는 persist 때 INSERT 전에 DB 시퀀스 객체에서 nextval 로 받아 온다. 1차 캐시의 키가 id 이므로 IDENTITY 에서는 Hibernate 가 persist 순간 INSERT 를 즉시 실행하고 getGeneratedKeys 로 id 를 받아 온다 → 그 엔티티는 쓰기 지연되지 않고 INSERT 배치도 꺼진다. 100건 저장 시 IDENTITY 는 왕복 100번, SEQUENCE + allocationSize=50 + batch_size=50 은 nextval 2번 + 배치 2번. 요청당 INSERT 가 적은 서비스는 단순성과 이식성 때문에 IDENTITY 로 충분하고, 대량 적재가 늘면 SEQUENCE + batch_size 를 함께 바꾸거나 그 부분만 JdbcTemplate 배치로 처리한다.
contributors: [dongju]
source_refs:
  - https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#identifiers-generators
  - https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#batch
updated: 2026-10-17
---

# 식별자 생성 전략 — IDENTITY vs SEQUENCE

## 개념 정의

| | 누가 | 언제 |
|---|---|---|
| `IDENTITY` | DB (identity / auto_increment 컬럼) | **INSERT 를 실행해야** 생김 |
| `SEQUENCE` | DB (시퀀스 객체) | `persist` 때 **INSERT 전에** `nextval` 로 받아 옴 |

```
-- SEQUENCE
new Post(...)          → id = null (DB 와 무관)
em.persist(post)       → select nextval('post_seq')   ← 번호를 먼저 받아 옴
                       → post.id = 1 → 1차 캐시 저장 (INSERT 는 아직)
커밋 직전 플러시        → insert into post (id, ...) values (1, ...)
```

- 시작값·증가값은 엔티티가 아니라 **DB 시퀀스 객체의 속성**이다.

## 작동 원리

### IDENTITY 는 persist 순간 INSERT 가 나간다

```
1차 캐시는 id 를 키로 엔티티를 저장한다 ([[persistence-context]])
  → IDENTITY 는 persist 순간 id 를 모른다 → 키가 없어 캐시에 넣을 수 없다
  → Hibernate 는 persist 순간 INSERT 를 즉시 실행
  → JDBC getGeneratedKeys() 로 DB 가 만든 id 를 받아 엔티티에 세팅 → 1차 캐시 저장
```

```java
em.persist(post);                   // INSERT 즉시 실행
System.out.println(post.getId());   // 값이 찍힌다
```

> **IDENTITY 에서는 그 엔티티의 INSERT 에 쓰기 지연이 동작하지 않는다.**

- INSERT 가 **나갔다**고 커밋된 건 아니다 — 이후 예외면 롤백된다 ([[jpa-flush]]). 다만 소비된 id 는 돌아오지 않아 **구멍**이 생긴다.

### JDBC 배치도 안 된다

```yaml
spring.jpa.properties.hibernate.jdbc.batch_size: 50
```

- 배치는 쓰기 지연 저장소에 **쌓인** INSERT 를 묶어 한 번의 왕복으로 보낸다.
- IDENTITY 는 쌓일 틈이 없다 → Hibernate 가 IDENTITY 엔티티의 **INSERT 배치를 끈다**.

### SEQUENCE 는 nextval 도 아낀다 — allocationSize

```java
@SequenceGenerator(name = "post_seq_gen", sequenceName = "post_seq", allocationSize = 50)  // 기본 50
```

- DB 시퀀스를 50 씩 올려 두고, 그 범위의 번호는 **Hibernate 가 메모리에서** 1씩 나눠 준다(pooled optimizer). 다 쓰면 다시 `nextval`.
- 재시작하면 받아 두고 못 쓴 번호는 버려져 역시 **구멍**이 생긴다 → 어느 전략도 연속 id 를 보장하지 않는다.

| 100건 저장 | 왕복 |
|---|---|
| IDENTITY (+ batch_size 50) | INSERT **100번** |
| SEQUENCE + allocationSize 50 + batch_size 50 | nextval **2번** + INSERT 배치 **2번** ≈ 4번 |

## 트레이드오프 / 한계

**IDENTITY 를 고르는 이유**
- 요청 하나에 INSERT 가 한두 건인 서비스(게시판 등)에서는 즉시 INSERT·배치 불가 비용이 거의 없다.
- 설정이 단순하고, MySQL 처럼 **시퀀스가 없는 DB** 로 옮겨도 그대로 동작한다.

**약점이 드러나는 곳**
- 단건 저장이 아니라 **대량 `saveAll`**(외부 일정 일괄 가져오기, 일괄 이월 등). 100건이면 왕복 100번.

**바꾼다면**
- 시퀀스를 지원하는 DB(PostgreSQL 등)라면 **SEQUENCE + `allocationSize` + `jdbc.batch_size` 를 함께** 설정한다. ⚠️ batch_size 를 안 켜면 SEQUENCE 로 바꿔도 배치는 안 된다.
- 대량 적재가 일부뿐이면 IDENTITY 를 유지하고 그 부분만 **`JdbcTemplate.batchUpdate`**.
- 여러 엔티티 타입의 INSERT 가 섞이면 `hibernate.order_inserts` 로 배치 효율을 높인다.

> **면접 답변** — "IDENTITY는 DB가 INSERT 시점에 id를 만듭니다. 그런데 영속성 컨텍스트의 1차 캐시는 id를 키로 엔티티를 관리하기 때문에, Hibernate는 persist 순간 INSERT를 즉시 실행해서 id를 받아 옵니다. 그래서 그 엔티티는 쓰기 지연이 되지 않고, INSERT를 묶어 보내는 JDBC 배치도 쓸 수 없습니다. 요청 하나에 INSERT가 한두 건인 서비스라 이 비용이 거의 없고, 설정이 단순하며 시퀀스가 없는 DB에서도 동작해서 IDENTITY를 골랐습니다. 대량 적재가 늘어나면 시퀀스로 바꾸고 allocationSize와 jdbc.batch_size를 함께 설정하거나, 그 부분만 JdbcTemplate 배치로 처리하겠습니다."

> [!WARNING]
> **오답 코너**
> - **"SEQUENCE 는 엔티티 생성 시에 값이 만들어진다"** — `new` 는 DB 와 무관. **persist 시점**에 시퀀스 객체에서 `nextval`.
> - **"DB 에 저장돼야 id 가 생겨서 쓰기 지연이 안 된다"** — 중간 고리 **"1차 캐시의 키가 id"** 가 빠지면 인과가 성립하지 않는다.
> - **"미리 받은 번호를 어노테이션이 나눠 쓴다"** — **Hibernate(pooled optimizer)** 가 메모리에서.
> - **"약점은 단건 저장에서 드러난다"** — 단건은 즉시든 지연이든 차이 없다. 비용이 되는 건 **대량 saveAll**.
> - **"대량 적재가 생기면 시퀀스로 바꾸면 된다"** — **batch_size 도 함께**.

## 복습 체크

- [ ] IDENTITY 와 SEQUENCE 에서 id 를 누가, 언제 만드는가?
- [ ] IDENTITY 에서 persist 순간 INSERT 가 나가는 이유를 1차 캐시 키와 연결해 설명할 수 있는가?
- [ ] persist 직후 getId() 는? 그 INSERT 는 롤백될 수 있는가?
- [ ] IDENTITY 에서 JDBC 배치가 꺼지는 이유는?
- [ ] allocationSize 가 nextval 왕복을 줄이는 원리와 그 대가는?
- [ ] 100건 저장 시 두 전략의 왕복 수를 비교할 수 있는가?
- [ ] IDENTITY 를 골라도 되는 조건과, 바꿀 때 함께 바꿔야 하는 설정은?

## 관련

[[persistence-context]] · [[jpa-flush]] · [[jpa-bulk-update-and-clear]] · [[jpa-entity-lifecycle-merge]] · [[wrapper-cache-autoboxing]]
