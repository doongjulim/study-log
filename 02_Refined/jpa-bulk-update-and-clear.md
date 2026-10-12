---
type: refined
slug: jpa-bulk-update-and-clear
tags: [벌크 연산, bulk update, bulk operation, 벌크 업데이트, executeUpdate, Modifying, "@Modifying", clearAutomatically, flushAutomatically, em.clear, clear, 영속성 컨텍스트 초기화, 준영속, detached, 1차 캐시, first-level cache, stale data, 낡은 데이터, 데이터 불일치, 배치 처리, batch processing, 대량 처리, flush 후 clear, flush and clear, hibernate.jdbc.batch_size, JDBC batch, 메모리, OutOfMemoryError, 동일성 보장, 스냅샷]
topic: 벌크 연산이 1차 캐시를 낡게 만드는 문제와 em.clear() 해법, clear 의 부작용(준영속), 대량 배치의 flush → clear 패턴
summary: 벌크 UPDATE 는 DB 만 직접 고치므로 이미 1차 캐시에 있는 엔티티는 옛 값이고, 이후 find() 는 DB 를 안 가 낡은 값을 돌려준다. 플러시(방향 반대)나 커밋(원자성 파괴)이 아니라 em.clear() 또는 @Modifying(clearAutomatically = true) 로 비운다. clear 후 기존 참조는 준영속이 되어 동일성 보장과 변경 감지가 끊긴다. 대량 배치는 N건마다 flush 후 clear 하며 순서가 바뀌면 변경이 증발한다.
contributors: [dongju]
source_refs:
  - https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html
  - https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html
updated: 2026-10-12
---

# 벌크 연산과 `em.clear()`

## 개념 정의

- **벌크 연산**: `update ... set ...` / `delete ...` JPQL 을 `executeUpdate()` 로 실행해 여러 행을 **DB 에서 직접** 한 번에 바꾸는 것. 영속성 컨텍스트를 거치지 않는다.
- **`em.clear()`**: 영속성 컨텍스트(1차 캐시 + 스냅샷 + 쓰기 지연 저장소)를 **통째로 비운다**. 트랜잭션은 유지된다.

## 작동 원리

### 1. 벌크 연산이 만드는 불일치

```java
@Transactional
public void test() {
    Member a = em.find(Member.class, 1L);   // DB age = 20, 1차 캐시에 올라감

    em.createQuery("update Member m set m.age = m.age + 1")
      .executeUpdate();                      // 직전 자동 플러시는 됨

    Member b = em.find(Member.class, 1L);
    System.out.println(b.getAge());          // 20   ← DB 는 21
}
```

- 벌크 연산도 JPQL 이라 실행 **직전** 플러시는 된다 ([[jpa-flush]]). 문제는 **이후**.
- 벌크 UPDATE 는 DB 만 고친다. 1차 캐시 안의 엔티티는 그대로 20.
- 두 번째 `find()` 는 1차 캐시에 id=1 이 있으니 DB 에 안 감 → **낡은 엔티티** 반환. 평소 장점인 1차 캐시가 여기선 원인.

### 2. 해결 — 방향을 맞춘다

| 시도 | 결과 |
|---|---|
| `em.flush()` | ✗ 방향 반대(컨텍스트 → DB). 1차 캐시의 20 그대로 |
| 커밋 | △ 캐시는 사라지지만 **메서드 중간에 트랜잭션이 끊겨 원자성 파괴** |
| **`em.clear()`** | ✓ 컨텍스트 비움, 트랜잭션 유지 |
| **`@Modifying(clearAutomatically = true)`** | ✓ Spring Data JPA 벌크 쿼리 직후 자동 clear |

```java
em.createQuery("update Member m set m.age = m.age + 1").executeUpdate();
em.clear();
Member b = em.find(Member.class, 1L);   // 캐시 비었음 → SELECT → 21 을 가진 새 엔티티
```

다른 전략: 벌크 연산을 1차 캐시가 비어 있는 **트랜잭션 맨 앞**에서 실행.

### 3. clear 의 부작용 — 준영속

```java
@Transactional
public void test() {
    Member a = em.find(Member.class, 1L);   // age = 20
    em.clear();
    Member b = em.find(Member.class, 1L);

    System.out.println(a == b);   // false
    a.setAge(99);                 // DB 반영 안 됨
}
```

- `a == b` **false** — b 는 DB 에서 새로 읽은 **다른 인스턴스**. 동일성 보장은 같은 컨텍스트 안에서만 ([[persistence-context]]).
- `a.setAge(99)` **반영 안 됨** — a 의 스냅샷이 사라져 비교 대상이 없다 → 변경 감지 불가 ([[dirty-checking]]).
- 컨텍스트에서 떨어져 관리받지 못하는 엔티티 = **준영속(detached)**. 멀쩡한 자바 객체지만 JPA 에겐 남남.

### 4. 대량 배치 — N건마다 flush → clear

엔티티마다 **엔티티 + 스냅샷**이 쌓이므로 10만 건을 한 트랜잭션에서 수정하면 메모리가 터질 수 있다.

```java
for (int i = 0; i < members.size(); i++) {
    // ... 수정
    if (i % 500 == 0) {
        em.flush();   // ① 지금까지의 변경을 DB 로
        em.clear();   // ② 그 다음 1차 캐시·스냅샷 비워 메모리 회수
    }
}
```

- **순서가 바뀌면(clear → flush)**: 수정한 엔티티가 전부 준영속 + 쓰기 지연 저장소도 비워짐 → flush 할 게 없음 → **변경 증발**.
- **매 건마다** flush 하면 쓰기 지연의 이점(모아 보내기)이 사라지고 DB 왕복이 건수만큼. 실무는 **100~1000건 단위**로, `hibernate.jdbc.batch_size` 와 맞춰 JDBC 배치 전송.

## 트레이드오프 / 한계

- `clear()` 는 **통째로** 비운다 → 계속 쓰려던 엔티티 참조도 준영속이 된다. 필요하면 다시 조회해야 한다. (특정 엔티티만 떼려면 `em.detach(entity)`)
- 배치 크기: 메모리 사용량 ↔ DB 왕복 횟수의 트레이드오프.
- 진짜 대량 처리는 영속성 컨텍스트 자체가 부담 → `StatelessSession`, JdbcTemplate `batchUpdate`, Spring Batch chunk 도 검토.

> [!WARNING]
> **오답 코너**
> - **"벌크 연산 후 낡은 캐시는 플러시로 해결"** — 플러시는 컨텍스트 → DB 방향. 낡은 건 컨텍스트 쪽이라 **비워야(clear)** 한다.
> - **"커밋하면 된다"** — 캐시는 사라지지만 트랜잭션 중간 확정으로 **원자성이 깨진다**. 캐시 갱신하려고 "전부 성공 or 전부 롤백"을 포기하는 셈.
> - **"배치는 clear → flush"** — **flush → clear**. clear 를 먼저 하면 스냅샷·쓰기 지연 SQL 이 같이 날아가 변경이 증발한다.
> - **"한 건마다 flush → clear"** — 순서는 맞지만 너무 잦다. N건 단위로 묶어 JDBC 배치와 맞춘다.
> - 모르는 걸 추측으로 연달아 던지지 말 것 — "이름은 기억 안 나지만 1차 캐시를 비워 DB 에서 다시 읽게 해야 한다"처럼 **원리로** 답한다.

## 복습 체크

- [ ] 벌크 UPDATE 후 같은 id `find()` 의 출력값과 DB 값이 다른 이유를 1차 캐시로 설명할 수 있는가?
- [ ] 플러시와 커밋이 해법이 아닌 이유를 각각 말할 수 있는가?
- [ ] `em.clear()` 와 `@Modifying(clearAutomatically = true)` 를 댈 수 있는가?
- [ ] clear 전 참조 `a` 와 clear 후 `b` 의 `==` 결과와, `a` 를 수정했을 때의 결과는?
- [ ] 대량 배치에서 flush 와 clear 의 순서와 각각의 역할은? 매 건마다 하면 안 되는 이유는?

## 관련

[[persistence-context]] · [[jpa-flush]] · [[dirty-checking]] · [[open-in-view-osiv]] · [[lazy-loading-proxy]]
