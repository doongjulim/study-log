---
date: 2026-10-12
type: raw
author: dongju
tags: [변경 감지, dirty checking, 더티 체킹, 스냅샷, snapshot, 1차 캐시, first-level cache, 플러시, flush, UPDATE SQL, 전체 컬럼 업데이트, all columns update, DynamicUpdate, 동적 업데이트, SQL 캐시, 실행 계획 캐시, execution plan cache, PreparedStatement, JDBC 배치, JDBC batch, hibernate.jdbc.batch_size, 부트 시점 SQL 생성, Hibernate, JPA]
topic: 변경 감지가 UPDATE 를 만드는 흐름과, Hibernate 가 바뀐 컬럼이 아니라 전체 컬럼을 UPDATE 하는 이유(@DynamicUpdate 와의 트레이드오프)
summary: 조회 시 1차 캐시에 엔티티와 스냅샷이 함께 저장되고, 플러시(커밋 직전·JPQL 직전) 때 스냅샷과 비교해 UPDATE 를 만든다. 기본 UPDATE 는 id 를 뺀 전체 컬럼을 SET 하는데, SQL 모양이 하나라 애플리케이션 시작 시 미리 만들어 재사용할 수 있고 DB 실행 계획 캐시와 JDBC 배치에도 유리하기 때문이다. 바뀐 컬럼만 보내려면 @DynamicUpdate.
source: session
distilled: false
---

## 배운 개념

### 1. 변경 감지 흐름

```java
@Transactional
public void changeName(Long id) {
    Member m = em.find(Member.class, id);   // Member: id, name, age, email, address ... (컬럼 10개)
    m.setName("새이름");
}
```

```
em.find()   → 1차 캐시(저장소)에 엔티티 + 스냅샷 저장
setName()   → 순수 자바 필드 대입. SQL 안 생김, Hibernate 코드도 안 돎
플러시       → (커밋 직전 / JPQL 실행 직전)
              영속 엔티티를 스냅샷과 비교 → 바뀐 엔티티의 UPDATE 를 쓰기 지연 저장소에 등록 → DB 전송
커밋         → 확정
```

- 1차 캐시는 **저장되는 것**이 아니라 **저장소**. 그 안에 엔티티와 스냅샷이 들어간다.

### 2. 실제로 나가는 UPDATE — 전체 컬럼

```sql
update member set name=?, age=?, email=?, address=?, ... where id=?
```

- `name` 하나만 바꿔도 **id 를 뺀 모든 컬럼**을 SET. id 는 WHERE 에서 대상을 찾는 키.

### 3. 왜 전체 컬럼인가

**(A) Hibernate 쪽 — SQL 을 미리 만들어 둘 수 있다**

- 전체 컬럼 UPDATE 는 엔티티마다 모양이 하나 → **애플리케이션 시작(부트) 시** 엔티티 메타데이터를 읽으며 INSERT/UPDATE/DELETE SQL 을 미리 생성·캐시.
- 바뀐 컬럼만 보내면 컬럼 10개 기준 조합 2¹⁰ − 1 = **1023가지** → 미리 못 만들고 플러시마다 SQL 문자열을 새로 생성해야 함.

**(B) DB 쪽 — 같은 SQL 은 싸다**

- DB 는 SQL 마다 파싱 + 실행 계획 수립. **글자 하나 다르지 않은 같은 SQL**(`?` 자리표시자 = `PreparedStatement`)이면 파싱 결과·실행 계획 캐시 재사용.
- **JDBC 배치**도 SQL 모양이 같아야 여러 UPDATE 를 한 번에 묶어 보낼 수 있음 → `hibernate.jdbc.batch_size` 와 연결 ([[2026-10-12-jpa-bulk-update-clear-detached]]).

### 4. 대안 — `@DynamicUpdate`

```java
@Entity
@DynamicUpdate
public class Member { ... }
```

- 바뀐 컬럼만 SET. 대신 플러시마다 SQL 생성, 캐시·배치 이점 감소.
- 쓸 때: 컬럼이 아주 많거나 큰 컬럼(TEXT, BLOB)이 있어 전송량 절감 이득이 SQL 재생성 비용보다 클 때.

### 5. 면접 답변 뼈대

> "조회할 때 영속성 컨텍스트가 엔티티와 함께 스냅샷을 저장해 두고, 플러시 시점에 둘을 비교해 바뀐 엔티티만 UPDATE 합니다. 이때 Hibernate 는 기본적으로 바뀐 컬럼이 아니라 전체 컬럼을 UPDATE 하는데, SQL 모양이 하나라 애플리케이션 시작 시 미리 만들어 재사용할 수 있고, DB 실행 계획 캐시와 JDBC 배치에도 유리하기 때문입니다. 컬럼이 많거나 큰 컬럼이 있으면 `@DynamicUpdate` 로 바뀐 컬럼만 보내게 할 수 있습니다."

## 오답·헷갈린 점

- **전**: 조회 시 "스냅샷과 **1차 캐시**가 저장됨" → **후**: 1차 캐시는 저장소. 그 안에 **엔티티 + 스냅샷**.
- **전**: 비교 시점 = "**트랜잭션 종료 시**" → (재질문) "플러시가 될 때" → **후**: 플러시 때. 플러시는 **커밋 직전 + JPQL 실행 직전**. JPQL 쪽을 또 빠뜨림.
- **전**: UPDATE SET 에 `id="id"` 포함 → **후**: id 는 WHERE 용, SET 에서 제외.
- **전**: 전체 컬럼 UPDATE 이유 = "**더 합리적이기 때문에**" → **후**: 구체적으로 ① 부트 시 SQL 미리 생성·재사용 ② DB 실행 계획 캐시 ③ JDBC 배치.
- **전**: SQL 을 만드는 시점 = "프로젝트가 **컴파일**되어 서버에서 실행될 때" → **후**: 컴파일이 아니라 **애플리케이션 시작(부트) 시점**.
- ✅ 처음 보는 질문(SET 절 컬럼)에서 "전체 컬럼이 들어간다"를 추론으로 맞힘.
- ⚠️ 반복 테마: **추상적인 인과 표현**("합리적이다") → 무엇이 구체적으로 싸지는지까지 말하기.

## Q&A

- **Q. 조회 시 1차 캐시에 저장되는 두 가지?** A. 엔티티, 스냅샷.
- **Q. 비교는 언제?** A. 플러시 때 (커밋 직전, JPQL 직전).
- **Q. name 만 바꾸면 UPDATE SET 은?** A. id 를 뺀 전체 컬럼.
- **Q. 왜 전체 컬럼인가 — Hibernate 쪽?** A. 모양이 하나라 부트 시 미리 생성·캐시. 바뀐 컬럼만이면 1023가지 조합이라 매번 생성.
- **Q. DB 쪽?** A. 같은 SQL 의 파싱·실행 계획 캐시 재사용, JDBC 배치 가능.
- **Q. 바뀐 컬럼만 보내려면?** A. `@DynamicUpdate`.

## 다음 학습

- `show-sql` 로 기본 UPDATE 와 `@DynamicUpdate` UPDATE 로그 직접 비교
- 전체 컬럼 UPDATE 와 동시 수정 충돌: 서로 다른 컬럼을 동시에 수정하면 한쪽이 덮어써지는 문제 → `@Version` 낙관적 락
- 이어지는 노트: [[2026-10-12-dirty-checking-cost-readonly]]
