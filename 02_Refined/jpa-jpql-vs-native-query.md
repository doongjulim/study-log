---
type: refined
slug: jpa-jpql-vs-native-query
tags: [JPQL, 객체쿼리, object-query, NativeQuery, nativeQuery, 네이티브쿼리, SQL, Dialect, 방언, H2Dialect, PostgreSQLDialect, Hibernate, JDBC드라이버, DB독립성, database-independence, insert-select, 벌크insert, bulk-insert, IDENTITY, 생성된id, CreatedDate, Modifying, clearAutomatically, 1차캐시, 파생쿼리, derived-query, 쿼리선택기준, JPA, 면접답변]
topic: JPQL 은 엔티티·필드 대상 객체 쿼리로 Hibernate 가 Dialect 로 번역해 DB 독립적이라 기본값이고, Native 는 테이블·컬럼 SQL 그대로라 JPQL 로 표현할 수 없거나 막힐 때만 쓴다
summary: JPQL 의 from Post p, p.author.nickname 은 엔티티와 필드 이름이고, JDBC 드라이버가 아니라 Hibernate 가 매핑 정보와 Dialect 로 DB 에 맞는 SQL 로 번역하므로 DB 를 바꿔도 그대로 동작한다. Native 는 테이블·컬럼을 직접 쓰는 SQL 그대로라 DB 에 묶이므로, JPQL 로 표현할 수 없거나 막힐 때만 쓴다. 예: IDENTITY 엔티티를 회원 수만큼 넣는 알림은 save() 반복이면 왕복이 N 번이고 JPQL insert-select 는 생성된 id 를 돌려받을 수 없어 Hibernate 가 막으므로 Native insert-select 한 문장으로 처리하고, 벌크 연산이라 clearAutomatically 로 1차 캐시를 비운다.
contributors: [dongju]
source_refs:
  - https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#query-language
  - https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html
updated: 2026-10-18
---

# JPQL vs Native Query

## 개념 정의

```java
// JPQL — 엔티티·필드 대상
"select new com.example.board.post.dto.PostSummary(p.id, p.title, p.author.nickname, ...) from Post p"

// Native — 테이블·컬럼 대상
"insert into notification (recipient_id, message, url, read_flag, created_at) select m.id, ... from member m ..."
```

> **JPQL** = 엔티티와 필드를 대상으로 쓰는 **객체 쿼리**. Hibernate 가 번역 → **DB 독립적**.
> **Native** = 테이블과 컬럼을 직접 쓰는 **SQL 그대로**. 번역 없음 → **DB 에 묶임**.

## 작동 원리

```
JPQL  "select p from Post p where p.author.id = :id"
  │  Hibernate 가 엔티티 매핑 정보 + Dialect(H2Dialect / PostgreSQLDialect)로 번역
  ▼
SQL   "select p.id, p.title, ... from post p where p.author_id = ?"
  │  JDBC 드라이버가 DB 로 전송 (번역하지 않는다)
  ▼
DB
```

- Dialect 가 페이징 문법, 함수 이름 등 DB 별 차이를 맞춘다 → 개발 DB(H2) → 운영 DB(PostgreSQL) 이전에도 JPQL 은 그대로.
- JPQL 은 1차 캐시를 거치지 않고 DB 로 가므로 실행 **직전 자동 플러시**가 일어난다 ([[jpa-flush]]).

### 선택 기준

| 방식 | 언제 |
|---|---|
| 파생 쿼리 (`findBy…`) | 조건이 단순할 때 ([[derived-query-method]]) |
| JPQL `@Query` | **기본값** — 조인, DTO 조회, 정렬·집계 |
| Specification / Querydsl | 선택 조건이 여러 개 자유롭게 조합될 때 ([[jpa-specification-dynamic-query]]) |
| Native | **JPQL 로 표현할 수 없거나 막힐 때** — DB 전용 기능(upsert, 윈도 함수, 재귀 쿼리), IDENTITY 벌크 insert 등 |

### 사례 — 회원 수만큼 알림 넣기

```java
@Modifying(clearAutomatically = true)
@Query(value = """
        insert into notification (recipient_id, message, url, read_flag, created_at)
        select m.id, :message, :url, false, :createdAt
          from member m
         where m.id in (:recipientIds)
        """, nativeQuery = true)
int insertForAll(...);
```

| 시도 | 결과 |
|---|---|
| `save()` N 번 | INSERT 왕복 **N 번** (IDENTITY 는 persist 즉시 INSERT, 배치 불가 — [[jpa-id-generation-strategy]]) |
| JPQL `insert ... select` | Hibernate 가 **막음** — IDENTITY 엔티티는 생성된 id 를 받아 1차 캐시에 넣어야 하는데 벌크로는 돌려받을 수 없음 |
| Native `insert ... select` | ✅ 한 문장 (이 기능은 id 가 필요 없음) |

- `@CreatedDate` 는 엔티티를 거칠 때만 채워지므로 `created_at` 을 직접 넣는다.
- 벌크 연산이라 영속성 컨텍스트를 거치지 않는다 → `clearAutomatically = true` 로 **1차 캐시를 비워** 뒤따르는 조회가 DB 를 다시 읽게 한다 ([[jpa-bulk-update-and-clear]]).

## 트레이드오프 / 한계

- Native 는 DB 를 바꾸면 다시 써야 할 수 있다(문법·함수·예약어).
- Native 결과는 엔티티 매핑이나 DTO/인터페이스 프로젝션으로 받아야 하고, 쓰기라면 영속성 컨텍스트와 어긋나므로 clear 등 뒷정리가 필요하다.
- JPQL 도 문자열이라 오타는 구동 시점(`@Query` 검증)에 드러난다 — 파생 쿼리와 같은 fail-fast.

> **면접 답변** — "기본은 JPQL입니다. 엔티티와 필드를 대상으로 쓰고, Hibernate가 Dialect에 맞춰 SQL로 번역해 주니 DB를 바꿔도 그대로 동작합니다. Native는 JPQL로 표현할 수 없거나 막힐 때만 씁니다. 회원 수만큼 알림을 넣는 기능에서 save()를 반복하면 IDENTITY라 회원 만 명에 INSERT 왕복이 만 번이고, JPQL insert-select는 Hibernate가 IDENTITY 엔티티에 막아서 Native insert-select 한 문장으로 처리했습니다. 대신 DB에 묶이고 영속성 컨텍스트를 거치지 않으니 clearAutomatically를 붙였습니다."

> [!WARNING]
> **오답 코너**
> - **"JPQL 을 SQL 로 바꿔 주는 건 DB 커넥터"** — **Hibernate** 가 Dialect 로 번역. JDBC 드라이버는 전송만.
> - **"clearAutomatically 는 DB 실행이 끝난 뒤 트랜잭션을 종료하게 한다"** — 벌크 **후에 1차 캐시를 비워** 다음 조회가 DB 를 읽게 한다.
> - **"Native 는 다량 저장할 때 쓴다 / JPQL 로 하면 1만 번 왕복"** — 1만 번 왕복은 **save() 반복**. JPQL 은 IDENTITY 때문에 **막힘**. Native 의 기준은 "JPQL 로 표현 불가하거나 막힐 때".

## 복습 체크

- [ ] JPQL 의 `Post`, `author` 는 무엇의 이름인가? Native 는 무엇을 쓰는가?
- [ ] JPQL 을 SQL 로 바꾸는 주체와, DB 를 바꿔도 동작하는 이유는?
- [ ] 파생 쿼리 / JPQL / Specification / Native 를 각각 언제 쓰는가?
- [ ] IDENTITY 엔티티를 대량으로 넣을 때 save 반복·JPQL·Native 각각의 결과는?
- [ ] Native 쓰기 쿼리에 clearAutomatically 를 붙이는 이유는?

## 관련

[[derived-query-method]] · [[jpa-specification-dynamic-query]] · [[jpa-id-generation-strategy]] · [[jpa-bulk-update-and-clear]] · [[jpa-flush]] · [[jpa-dto-projection]]
