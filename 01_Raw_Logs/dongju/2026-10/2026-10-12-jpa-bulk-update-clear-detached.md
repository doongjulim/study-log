---
date: 2026-10-12
type: raw
author: dongju
tags: [벌크 연산, bulk update, bulk operation, executeUpdate, Modifying, clearAutomatically, flushAutomatically, em.clear, clear, 준영속, detached, 영속성 컨텍스트 초기화, 1차 캐시, first-level cache, stale data, 낡은 데이터, 데이터 불일치, 배치 처리, batch processing, flush 후 clear, hibernate.jdbc.batch_size, JDBC batch, 메모리, OutOfMemoryError, 동일성 보장, 스냅샷]
topic: 벌크 연산은 영속성 컨텍스트를 우회해 1차 캐시를 낡게 만든다 — 해결은 clear(), 그 부작용은 준영속. 대량 배치는 N건마다 flush → clear
summary: 벌크 UPDATE 는 DB 만 직접 고치므로 이미 1차 캐시에 있는 엔티티는 옛 값 그대로이고, 이후 find() 는 DB 를 가지 않아 낡은 값을 돌려준다. 플러시(컨텍스트→DB)나 커밋(원자성 파괴)이 아니라 em.clear() 또는 @Modifying(clearAutomatically = true) 로 비워야 한다. clear 후 기존 참조는 준영속이 되어 동일성 보장과 변경 감지가 끊긴다. 10만 건 배치는 N건마다 flush 후 clear 해야 하며 순서가 바뀌면 변경이 증발한다.
source: session
distilled: false
---

## 배운 개념

### 1. 벌크 연산이 만드는 불일치

```java
@Transactional
public void test() {
    Member a = em.find(Member.class, 1L);   // DB age = 20, 1차 캐시에 올라감

    em.createQuery("update Member m set m.age = m.age + 1")
      .executeUpdate();                      // 벌크 UPDATE (직전 자동 플러시는 됨)

    Member b = em.find(Member.class, 1L);
    System.out.println(b.getAge());          // 20  ← DB 는 21
}
```

- 벌크 연산도 JPQL 이라 실행 **직전** 플러시는 된다. 문제는 **이후**.
- 벌크 UPDATE 는 DB 를 직접 고칠 뿐, 영속성 컨텍스트 안의 엔티티는 안 고친다.
- 두 번째 `find()` 는 1차 캐시에 id=1 이 있으니 DB 를 안 가고 **낡은 엔티티(20)** 반환.
- 원인 = **1차 캐시** (평소엔 장점이 여기선 독).

### 2. 해결 — 방향을 맞춰야 한다

| 시도 | 결과 |
|---|---|
| `em.flush()` | ✗ 방향이 반대(컨텍스트 → DB). 1차 캐시의 20 은 그대로 |
| 커밋 | △ 캐시는 사라지지만 **메서드 중간에 트랜잭션이 끊겨 원자성 파괴**. 이후 예외가 나도 앞부분은 확정됨 |
| **`em.clear()`** | ✓ 영속성 컨텍스트를 통째로 비움. 트랜잭션은 유지 |
| **`@Modifying(clearAutomatically = true)`** | ✓ Spring Data JPA 벌크 쿼리 실행 직후 자동 clear |

```java
em.createQuery("update Member m set m.age = m.age + 1").executeUpdate();
em.clear();
Member b = em.find(Member.class, 1L);   // 1차 캐시 비었음 → SELECT → 21 을 가진 새 엔티티
```

### 3. clear 의 부작용 — 준영속(detached)

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

- `a == b` **false**: clear 로 1차 캐시가 비워져 b 는 DB 에서 새로 조회한 **다른 인스턴스**. 동일성 보장은 "같은 영속성 컨텍스트 안에서"만.
- `a.setAge(99)` **반영 안 됨**: a 의 스냅샷이 clear 로 사라져 비교 대상이 없음 → 변경 감지 불가.
- 이렇게 영속성 컨텍스트에서 떨어져 관리받지 못하는 엔티티 = **준영속(detached)**. 멀쩡한 자바 객체지만 JPA 입장에선 남남.

### 4. 대량 배치 — flush → clear, N건 단위

- 10만 건을 한 트랜잭션에서 조회·수정하면 엔티티마다 **엔티티 + 스냅샷**이 쌓여 메모리 폭발 가능.
- **flush** 로 지금까지의 변경을 DB 로 보내고 → **clear** 로 1차 캐시·스냅샷 비워 메모리 회수.

```java
for (int i = 0; i < members.size(); i++) {
    // ... 수정
    if (i % 500 == 0) {
        em.flush();   // ① 변경 내용 먼저 DB 로
        em.clear();   // ② 그 다음 메모리 회수
    }
}
```

- **순서가 바뀌면(clear → flush)**: 수정해 둔 엔티티가 전부 준영속 + 쓰기 지연 저장소도 비워짐 → flush 할 게 없음 → **변경 증발**.
- **매 건마다** flush 하면 쓰기 지연의 이점(모아 보내기)이 사라지고 DB 왕복 10만 번. 실무는 **100~1000건 단위**, `hibernate.jdbc.batch_size` 와 맞춰 JDBC 배치 전송.
- 트레이드오프: 메모리 사용량 ↔ DB 왕복 횟수.

## 오답·헷갈린 점

- **전**: 벌크 연산 후 낡은 1차 캐시 해결 = "**플러시**" → **후**: 플러시는 컨텍스트 → DB 방향이라 1차 캐시는 그대로. 필요한 건 **비우기(`clear`)**.
- **전**: (재도전) "**커밋**해줘야 함" → **후**: 캐시는 사라지지만 트랜잭션 중간 확정으로 **원자성 파괴**. 캐시 갱신하려고 "전부 성공 or 전부 롤백"을 포기하는 셈. 정답은 `em.clear()` / `@Modifying(clearAutomatically = true)`.
- **전**: 배치 순서 = **clear → flush** ("db 로 보내지 않았던 내용이 모두 보내진다") → **후**: **flush → clear**. clear 를 먼저 하면 스냅샷·쓰기 지연 SQL 이 같이 날아가 변경 증발. (바로 앞에서 "clear 하면 스냅샷이 지워져 변경 감지 안 된다"고 스스로 답한 것과 모순)
- **전**: "한 명마다 flush → clear" → **후**: 순서는 맞지만 너무 잦음. N건(100~1000) 단위.
- ⚠️ 힌트를 줬는데 감이 안 잡히자 **추측을 연달아 던짐**(플러시 → 커밋). 면접에선 "이름은 기억 안 나지만 1차 캐시를 비워서 DB 에서 다시 읽게 해야 한다"처럼 **원리로 답하기**.
- ⚠️ 플러시 방향 혼동이 여기서도 재발 (flush 노트와 같은 뿌리).

## Q&A

- **Q. 벌크 UPDATE 후 같은 id find → 출력값과 DB 값?** A. 20, DB 는 21.
- **Q. 원인 기능은?** A. 1차 캐시 — 이미 있으니 DB 를 안 감.
- **Q. 해결책?** A. `em.clear()` 또는 `@Modifying(clearAutomatically = true)`.
- **Q. clear 후 find 동작?** A. 1차 캐시가 비었으니 SELECT → 최신값 21 을 가진 새 엔티티를 1차 캐시에 올림.
- **Q. clear 전후 참조 `a == b`?** A. false. 다른 인스턴스.
- **Q. clear 전 참조 a 를 수정하면?** A. 스냅샷이 없어 변경 감지 안 됨 → 반영 안 됨 (준영속).
- **Q. 10만 건 배치 메모리 대책?** A. N건마다 flush → clear. 순서 중요.

## 다음 학습

- `@Modifying(flushAutomatically = true)` 의 의미와 clearAutomatically 와 함께 쓸 때
- 벌크 연산을 트랜잭션 맨 앞에 두는 전략 (1차 캐시가 비어 있을 때 실행)
- 대량 처리 대안: `StatelessSession`, JdbcTemplate batchUpdate, Spring Batch chunk
- `IDENTITY` 전략에서 JDBC 배치 insert 가 안 되는 이유
- 준영속 엔티티 다시 붙이기: `merge()` vs 다시 조회 후 변경 감지
