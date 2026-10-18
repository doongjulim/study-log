---
date: 2026-10-18
type: raw
author: dongju
tags: [JPQL, 객체 쿼리, object query, Native Query, nativeQuery, 네이티브 쿼리, SQL, Dialect, 방언, H2Dialect, PostgreSQLDialect, Hibernate, JDBC 드라이버, DB 독립성, database independence, insert select, 벌크 insert, bulk insert, IDENTITY, 생성된 id, CreatedDate, Modifying, clearAutomatically, 1차 캐시, 파생 쿼리, derived query, study-board, NotificationRepository, insertForAll, 면접 답변]
topic: JPQL 은 엔티티·필드 대상 객체 쿼리로 Hibernate 가 Dialect 로 SQL 번역해 DB 독립적이라 기본값, Native 는 테이블·컬럼 SQL 그대로라 JPQL 로 표현 불가하거나 막힐 때만 — study-board 의 유일한 Native insertForAll
summary: JPQL 의 from Post p, p.author.nickname 은 엔티티와 필드 이름이고, JDBC 드라이버가 아니라 Hibernate 가 매핑 정보와 Dialect 로 DB 에 맞는 SQL 로 번역하므로 H2→PostgreSQL 이전에도 그대로 동작한다. Native 는 테이블·컬럼을 직접 쓰는 SQL 그대로라 DB 에 묶이므로 JPQL 로 표현할 수 없거나 막힐 때만 쓴다. study-board 의 insertForAll 은 전체 공개 계획 알림을 회원 수만큼 넣는데, save() 반복은 IDENTITY 라 1만 번 왕복이고 JPQL insert-select 는 IDENTITY 엔티티라 Hibernate 가 막아(생성된 id 를 돌려받을 수 없음) Native insert-select 한 문장으로 처리하고, 벌크 연산이라 clearAutomatically 로 1차 캐시를 비운다.
source: session
distilled: true
distilled_at: 2026-10-18
---

## 배운 개념

### 1. study-board 분포

| 방식 | 개수 |
|---|---|
| 파생 쿼리 (`findBy…`) | 약 57 |
| JPQL `@Query` | 26 |
| Native | **1** (`NotificationRepository.insertForAll`) |
| Specification | **1곳** (`PlanSpecifications`) |

→ "기본은 JPQL(과 파생 쿼리), 예외만 나머지"가 코드에 드러남.

### 2. JPQL — 객체 쿼리

```java
"select new com.example.board.post.dto.PostSummary(p.id, p.title, p.author.nickname, ...) from Post p"
```

- `Post` = **엔티티(클래스) 이름**, `author` = **필드** 이름.
- 번역은 **Hibernate** 가 한다. JDBC 드라이버는 완성된 SQL 을 실어 나르기만.

```
JPQL  "select p from Post p where p.author.id = :id"
  │  Hibernate 가 매핑 정보 + Dialect(H2Dialect / PostgreSQLDialect)로 번역
  ▼
SQL   "select p.id, p.title, ... from post p where p.author_id = ?"
  │  JDBC 드라이버가 DB 로 전송
  ▼
DB
```

- Dialect 가 페이징 문법·함수 이름 등 DB 별 차이를 맞춘다 → H2 → PostgreSQL 이전에도 그대로 동작 → **DB 독립적**.

### 3. Native — SQL 그대로

```java
// NotificationRepository — study-board 의 유일한 Native
@Modifying(clearAutomatically = true)
@Query(value = """
        insert into notification (recipient_id, message, url, read_flag, created_at)
        select m.id, :message, :url, false, :createdAt
          from member m
         where m.id in (:recipientIds)
        """, nativeQuery = true)
int insertForAll(...);
```

- 테이블·컬럼(`notification`, `recipient_id`)을 직접 쓴다 → 번역 없음 → **DB 에 묶임**.
- **기준**: "다량 저장"이 아니라 **JPQL 로 표현할 수 없거나 막힐 때** (DB 전용 기능: upsert, 윈도 함수, 재귀 쿼리 등).

### 4. insertForAll 이 Native 인 이유

전체 공개 계획 하나 → 회원 수만큼 알림.

| 시도 | 결과 |
|---|---|
| `save()` 1만 번 | INSERT 왕복 **1만 번** (IDENTITY 는 persist 즉시 INSERT, 배치도 안 됨) |
| JPQL `insert ... select` | Hibernate 가 **막음** — IDENTITY 엔티티는 INSERT 마다 생성된 id 를 받아 1차 캐시에 넣어야 하는데 벌크로는 돌려받을 수 없음 |
| Native `insert ... select` | ✅ 한 문장. 이 기능은 id 가 필요 없음 (실시간 전송 대상만 뒤에서 다시 조회) |

- `@CreatedDate` 는 엔티티를 거칠 때만 채워지므로 `created_at` 을 직접 넣는다.
- 벌크 연산이라 영속성 컨텍스트를 거치지 않음 → `clearAutomatically = true` 로 **1차 캐시를 비워** 바로 뒤의 `findByRecipient_IdInAndCreatedAt` 이 DB 를 다시 읽게 함.

## 오답·헷갈린 점

- **전**: JPQL 을 SQL 로 바꿔 주는 건 "DB 커넥터" → **후**: **Hibernate** 가 Dialect 로 번역. JDBC 드라이버는 전송만.
- **전**: `clearAutomatically` = "DB 의 실행이 완전히 끝난 후 트랜잭션이 종료할 수 있도록" → **후**: 벌크 **후에 1차 캐시를 비워** 다음 조회가 DB 를 다시 읽게 함. ⚠️ 10/12·10/14 에 배운 내용 재혼동 — flush/clear 방향 표 복습.
- **전**: 최종 답변에서 "Native 는 다량 저장 시 사용, JPQL 로 하면 1만 번 왕복" → **후**: 1만 번 왕복은 **save() 반복**. JPQL 은 IDENTITY 때문에 **막힘**. Native 의 기준은 "JPQL 로 표현 불가하거나 막힐 때". 대가(DB 종속, clear 필요)도 함께 말할 것.
- ✅ JPQL=클래스·필드 / Native=컬럼, IDENTITY 1만 번 왕복, 생성된 id 를 돌려받을 수 없음은 스스로 답함.

## Q&A

- **Q. JPQL 의 `Post`, `author` 는?** A. 엔티티 이름, 필드 이름.
- **Q. JPQL 을 SQL 로 바꾸는 주체?** A. Hibernate (Dialect).
- **Q. DB 를 바꿀 때 살아남는 쪽?** A. JPQL.
- **Q. insertForAll 이 save() 반복이 아닌 이유?** A. IDENTITY → 1만 번 왕복.
- **Q. JPQL insert-select 가 안 된 이유?** A. IDENTITY 엔티티 벌크 insert 는 생성된 id 를 돌려받을 수 없어 Hibernate 가 막음.
- **Q. clearAutomatically 를 붙인 이유?** A. 벌크 후 1차 캐시를 비워 다음 조회가 DB 를 읽게.

## 다음 학습

- Hibernate 6 HQL `insert ... values` / `insert ... select` 지원 범위와 IDENTITY 제약 직접 실행해 확인
- Native 결과를 DTO / 인터페이스 프로젝션으로 받는 방법
- 연결: [[jpa-id-generation-strategy]], [[jpa-bulk-update-and-clear]], [[jpa-dto-projection]], [[2026-10-18-specification-dynamic-query]]
