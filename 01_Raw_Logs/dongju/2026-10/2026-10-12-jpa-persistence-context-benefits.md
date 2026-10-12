---
date: 2026-10-12
type: raw
author: dongju
tags: [영속성 컨텍스트, persistence context, EntityManager, 엔티티 매니저, JPA, Hibernate, 1차 캐시, first-level cache, 동일성 보장, identity guarantee, 변경 감지, dirty checking, 스냅샷, snapshot, 쓰기 지연, write-behind, transactional write-behind, 트랜잭션 범위 영속성 컨텍스트, transaction-scoped persistence context, MyBatis 비교, 롤백, rollback, Transactional]
topic: 영속성 컨텍스트가 무엇이고 MyBatis 처럼 SQL 을 직접 날리는 방식과 비교해 무엇을 얻는가 — 롤백은 영속성 컨텍스트의 장점이 아니다
summary: 영속성 컨텍스트는 JPA 스펙이 정의하는 엔티티 저장 공간이며 EntityManager 로 접근하고, 스프링은 그 생명주기를 트랜잭션 범위에 묶어 관리한다. 본질은 SQL 이 아니라 엔티티와 스냅샷을 붙잡고 있는 것이고, 그 덕분에 1차 캐시·동일성 보장·변경 감지·쓰기 지연을 얻는다. 롤백은 DB 트랜잭션 기능이라 MyBatis 도 똑같이 누린다.
source: session
distilled: true
distilled_at: 2026-10-12
---

## 배운 개념

### 1. 정체 — 누가 정의하고 누가 관리하나

- **JPA 스펙**이 정의하는 개념. 스프링 없이 순수 Java + Hibernate 로도 존재한다.
- 접근 수단은 `EntityManager`. 스프링 없이 쓰면 직접 열고 닫아야 한다.
- 스프링은 **트랜잭션이 시작될 때 영속성 컨텍스트를 열고 끝날 때 닫는다** → "트랜잭션 범위 = 영속성 컨텍스트 범위". (OSIV 를 끄면 이 범위가 그대로 적용됨 → [[open-in-view-osiv]])

### 2. 롤백은 영속성 컨텍스트의 장점이 아니다

```java
@Transactional
public void order() {
    orderMapper.insert(order);      // INSERT 실행 성공
    stockMapper.decrease(itemId);   // 여기서 RuntimeException 발생
}
```

- MyBatis 여도 첫 INSERT 는 **DB 에 남지 않는다**. 쿼리가 DB 로 "날아간 것"과 "커밋된 것"은 다르다.
- 롤백은 **DB 트랜잭션 + `@Transactional`** 의 기능 → JPA 든 MyBatis 든 동일.
- 디테일: `@Transactional` 기본 롤백 대상은 **언체크 예외(RuntimeException, Error)**. 체크 예외는 커밋된다 → [[transactional-rollback-rules]]

### 3. 진짜로 얻는 것 — 같은 코드를 JPA / MyBatis 로 비교

```java
@Transactional
public void test() {
    Member a = em.find(Member.class, 1L);
    Member b = em.find(Member.class, 1L);

    System.out.println(a == b);   // ①
    a.setName("새이름");           // ②
}
```

| | JPA | MyBatis |
|---|---|---|
| SELECT 횟수 | **1번** — 두 번째는 1차 캐시에서 꺼냄 | 2번 — 호출마다 쿼리 |
| ① `a == b` | **true** — 같은 인스턴스 | false — 결과마다 새 객체 |
| ② update 호출 없이 DB 반영? | **바뀜** — 변경 감지 | 안 바뀜 — update 를 직접 호출해야 함 |

이름 붙이기:

- **1차 캐시**: 영속성 컨텍스트 내부에 `@Id → 엔티티` Map. 같은 id 재조회 시 DB 를 가지 않음.
- **동일성 보장**: 같은 영속성 컨텍스트 안에서 같은 id 는 항상 같은 인스턴스 (`==` 는 참조 주소 비교 → [[reference-vs-value-equality]]).
- **변경 감지**: 조회 시점의 스냅샷과 현재 값을 비교해 UPDATE 생성 → [[dirty-checking]]
- **쓰기 지연**: `persist()`/`remove()` 의 INSERT/DELETE 를 쓰기 지연 SQL 저장소에 모았다가 플러시 때 전송.

### 4. 네 가지 이점이 하나의 뿌리에서 나온다

- "바뀌었다"를 판단하려면 **비교 대상(스냅샷)** 이 있어야 한다.
- 비교하려면 조회한 엔티티를 반환하고 잊으면 안 되고 **붙잡고 있어야** 한다.
- 그 붙잡고 있는 저장소가 곧 1차 캐시 → 그래서 재조회 시 DB 생략(1차 캐시) + 같은 인스턴스 반환(동일성 보장)도 따라온다.

> 영속성 컨텍스트의 본질은 SQL 저장소가 아니라 **엔티티(와 스냅샷) 저장소**. SQL 은 그 결과로 만들어지는 부산물.

### 5. 면접 답변 뼈대

> "영속성 컨텍스트는 JPA 가 엔티티를 관리하는 저장 공간이고, 스프링에서는 트랜잭션 범위와 생명주기가 같습니다. 조회한 엔티티와 그 시점의 스냅샷을 붙잡고 있기 때문에, 같은 id 를 다시 조회하면 DB 에 가지 않고(1차 캐시) 같은 인스턴스를 돌려주며(동일성 보장), setter 로 값만 바꿔도 커밋 직전에 스냅샷과 비교해 UPDATE 를 만들어 줍니다(변경 감지). INSERT/DELETE 는 모아 두었다가 플러시 때 한 번에 보냅니다(쓰기 지연).
> 참고로 롤백은 영속성 컨텍스트가 아니라 DB 트랜잭션의 기능이라 MyBatis 도 똑같이 됩니다."

## 오답·헷갈린 점

- **전**: 영속성 컨텍스트는 "스프링에서 지원하는 기능" → **후**: **JPA 스펙**. 스프링은 생명주기(트랜잭션 범위)만 관리.
- **전**: 장점 = "MyBatis 는 롤백이 어렵지만 영속성 컨텍스트는 바로 롤백 가능" → **후**: 롤백은 **DB 트랜잭션 기능**. MyBatis + `@Transactional` 도 동일하게 롤백. (스스로 Q1-1 에서 "DB 에 남지 않는다"고 답하며 첫 답변의 전제가 무너짐)
- **전**: "수정·삭제·등록 시 모든 SQL 을 기록해 뒀다가 커밋 때 반영" → **후**: INSERT/DELETE 는 쓰기 지연 저장소에 쌓이는 게 맞지만, **UPDATE 는 미리 기록되지 않고** 플러시 때 스냅샷 비교로 생성. 본질은 엔티티 저장소.
- **전**: 같은 id 두 번 `find()` → SELECT **2번**, `a == b` **false** (MyBatis 와 동일하다고 생각) → **후**: 1차 캐시로 **1번**, 동일성 보장으로 **true**.
- ⚠️ 변경 감지(3번)는 맞혔는데 1, 2번은 틀림 → "변경 감지가 되려면 엔티티를 붙잡고 있어야 한다"는 연결이 처음엔 없었음. 힌트(Map 저장소) 후 바로 연결됨.
- ⚠️ 반복 테마: **DB 레벨 메커니즘(트랜잭션 롤백)과 JPA 레벨 메커니즘(영속성 컨텍스트)을 섞음**.

## Q&A

- **Q. 영속성 컨텍스트는 누가 정의하나?** A. JPA 스펙. 스프링은 트랜잭션 범위로 생명주기 관리.
- **Q. MyBatis + `@Transactional` 에서 중간 예외 시 앞선 INSERT 는?** A. 롤백되어 남지 않음. 롤백은 DB 트랜잭션 기능.
- **Q. 같은 id 두 번 find — SELECT 몇 번, `==` 는?** A. 1번, true.
- **Q. update 호출 없이 setter 만 → DB 반영?** A. JPA 는 반영(변경 감지), MyBatis 는 안 됨.
- **Q. JPA 는 변경을 어떻게 아나?** A. 조회 시점 스냅샷과 현재 엔티티 비교. 그래서 엔티티를 붙잡고 있어야 함.

## 다음 학습

- 엔티티 생명주기 4상태 정리: 비영속(new) / 영속(managed) / 준영속(detached) / 삭제(removed)
- `merge()` 동작과 준영속 엔티티 수정 시 주의점
- 이어지는 노트: [[2026-10-12-jpa-flush-vs-commit]], [[2026-10-12-jpa-bulk-update-clear-detached]]
