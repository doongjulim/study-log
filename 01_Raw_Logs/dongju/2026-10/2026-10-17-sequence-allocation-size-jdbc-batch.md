---
date: 2026-10-17
type: raw
author: dongju
tags: [JDBC 배치, JDBC batch, batch insert, hibernate.jdbc.batch_size, 배치 INSERT, 네트워크 왕복, round trip, IDENTITY 배치 불가, SEQUENCE, SequenceGenerator, allocationSize, pooled optimizer, nextval, saveAll, 대량 적재, bulk insert, JdbcTemplate.batchUpdate, order_inserts, PostgreSQL, MySQL, id 구멍, CalendarImportService, study-board, 트레이드오프, 면접 답변]
topic: IDENTITY 는 INSERT 를 즉시 실행하므로 JDBC 배치로 묶을 수 없다 — SEQUENCE + allocationSize + batch_size 와 왕복 수 비교, 그래도 IDENTITY 를 고른 이유와 바꿀 시점
summary: hibernate.jdbc.batch_size 는 쓰기 지연 저장소의 INSERT 를 묶어 한 번의 왕복으로 보내는데, IDENTITY 는 persist 마다 INSERT 를 즉시 실행해 묶을 기회가 없으므로 Hibernate 가 INSERT 배치를 끈다. 100건 저장 시 IDENTITY 는 왕복 100번, SEQUENCE + allocationSize=50 + batch_size=50 은 nextval 2번 + 배치 2번. study-board 는 요청당 INSERT 가 한두 건이라 단순성과 DB 이식성을 위해 IDENTITY 를 골랐고, 약점은 CalendarImportService 의 saveAll 같은 대량 적재에서 드러난다. 바꿀 땐 batch_size 도 함께 설정해야 하며, 일부만 JdbcTemplate.batchUpdate 로 처리하는 길도 있다.
source: session
distilled: false
---

## 배운 개념

### 1. JDBC 배치

```yaml
spring.jpa.properties.hibernate.jdbc.batch_size: 50
```

- 쓰기 지연 저장소에 모인 INSERT 를 플러시 때 **50개씩 묶어 한 번의 네트워크 왕복**으로 보낸다.
- 전제: INSERT 가 **쌓여 있어야** 묶을 수 있다.

### 2. IDENTITY 는 묶을 수 없다

- IDENTITY 는 persist 마다 INSERT 를 **즉시** 실행 → 쌓일 틈이 없다 → Hibernate 가 IDENTITY 엔티티의 **INSERT 배치를 끈다**.
- 설정을 켜도 100건이면 **100번 왕복**.

### 3. SEQUENCE 는 nextval 도 아낀다 — allocationSize

반격: "SEQUENCE 도 persist 마다 nextval 왕복이면 100번 아닌가?"

```java
@SequenceGenerator(name = "post_seq_gen", sequenceName = "post_seq", allocationSize = 50)  // 기본값 50
```

- DB 시퀀스 증가값을 50 으로 맞춰 두고, `nextval` 한 번에 받은 값을 기준으로 **50개 번호를 Hibernate 가 메모리에서** 1씩 나눠 준다 (pooled optimizer).
- 다 쓰면 다시 `nextval`.
- 100건: **nextval 2번 + INSERT 배치 2번 ≈ 왕복 4번** vs IDENTITY **100번**.
- 대가: 재시작하면 받아 두고 못 쓴 번호가 버려져 id 구멍.

### 4. 그래도 IDENTITY 를 고른 이유 (study-board)

- 게시판·플래너라 요청 하나에 INSERT 가 **한두 건** → 즉시 INSERT·배치 불가 비용이 거의 없다.
- 설정이 단순하고, MySQL 처럼 **시퀀스가 없는 DB** 로 옮겨도 그대로 동작한다.

### 5. 약점이 드러나는 곳

```java
// CalendarImportService — 외부 캘린더(.ics) 일정 일괄 가져오기
planRepository.saveAll(newPlans);   // 100개면 INSERT 왕복 100번
// PlanService — 미완료 계획 이월
planRepository.saveAll(unfinished.stream().map(plan -> plan.rolloverTo(to)).toList());
```

- 글 하나 저장은 단건이라 차이 없음. 비용이 되는 건 **대량 `saveAll`**.

### 6. 바꾼다면

- 운영 DB 후보 PostgreSQL 은 시퀀스 지원 → **SEQUENCE + `allocationSize` + `jdbc.batch_size`** 를 **함께** 설정.
  - ⚠️ study-board 에는 현재 `batch_size` 가 **설정돼 있지 않다** → SEQUENCE 로만 바꾸면 배치는 여전히 안 된다.
- 대량 적재가 한두 군데뿐이면 IDENTITY 유지 + 그 부분만 **`JdbcTemplate.batchUpdate`**.

### 7. 면접 답변 (30초)

> "IDENTITY는 DB가 INSERT 시점에 id를 만듭니다. 그런데 영속성 컨텍스트의 1차 캐시는 id를 키로 엔티티를 관리하기 때문에, Hibernate는 persist 순간 INSERT를 즉시 실행해서 id를 받아 옵니다. 그래서 그 엔티티는 쓰기 지연이 되지 않고, INSERT를 묶어 보내는 JDBC 배치도 쓸 수 없습니다.
> 그래도 study-board는 요청 하나에 INSERT가 한두 건인 게시판이라 이 비용이 거의 없고, 설정이 단순하며 MySQL처럼 시퀀스가 없는 DB로 옮겨도 그대로 동작해서 IDENTITY를 골랐습니다.
> 약점이 드러나는 곳은 외부 캘린더 일정을 한꺼번에 가져오는 saveAll이고, 일정 100개면 INSERT 왕복이 100번입니다. 이런 대량 적재가 늘어나면 PostgreSQL 시퀀스로 바꾸고 allocationSize와 jdbc.batch_size를 함께 설정하거나, 그 부분만 JdbcTemplate 배치로 처리하겠습니다."

## 오답·헷갈린 점

- **전**: 미리 받은 번호를 "어노테이션이" 1씩 증가시켜 사용 → **후**: **Hibernate(pooled optimizer)가 메모리에서** 나눠 준다.
- **전**: 약점이 드러나는 곳 = "post 저장 시 지연쓰기가 동작하지 않는다" → **후**: 단건 INSERT 는 즉시든 지연이든 차이 없음. 비용이 되는 건 **대량 `saveAll`**(캘린더 가져오기, 계획 이월).
- **전**: 대량 적재가 생기면 "시퀀스 전략으로 수정" → **후**: **`batch_size` 도 함께** 설정해야 배치가 된다(현재 미설정). 또는 그 부분만 `JdbcTemplate.batchUpdate`.
- ✅ SEQUENCE + batch 50 → 2번, IDENTITY → 100번, allocationSize 로 nextval 2번은 스스로 정확히 답함.

## Q&A

- **Q. SEQUENCE + batch_size 50, 100건 INSERT 왕복?** A. 2번 (+ nextval 2번).
- **Q. IDENTITY 면?** A. 100번. persist 마다 즉시 INSERT 라 묶을 수 없다.
- **Q. SEQUENCE 도 nextval 이 100번 아닌가?** A. allocationSize=50 이면 한 번에 50개 범위를 받아 메모리에서 나눠 쓴다 → 2번.
- **Q. 그래도 IDENTITY 를 고른 이유?** A. 요청당 INSERT 한두 건, 단순한 설정, 시퀀스 없는 DB 이식성.
- **Q. 약점이 드러나는 곳?** A. CalendarImportService / PlanService 의 대량 saveAll.

## 다음 학습

- `show-sql` + `hibernate.generate_statistics` 로 IDENTITY vs SEQUENCE `saveAll(100)` 쿼리 수 직접 비교
- `hibernate.order_inserts` / `order_updates`: 여러 엔티티 타입이 섞일 때 배치 효율
- id 를 URL 에 노출할 때의 열거 공격과 UUID·공개용 키 트레이드오프
- 연결: [[2026-10-17-identity-strategy-immediate-insert]], [[jpa-bulk-update-and-clear]], [[jpa-flush]]
