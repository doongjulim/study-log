---
type: refined
slug: persistence-context
tags: [영속성 컨텍스트, 영속성컨텍스트, persistence context, persistence-context, EntityManager, 엔티티 매니저, JPA, Hibernate, 1차 캐시, 1차캐시, first-level cache, first-level-cache, 동일성 보장, identity guarantee, 쓰기 지연, write-behind, transactional write-behind, 변경 감지, dirty checking, 스냅샷, snapshot, 트랜잭션 범위 영속성 컨텍스트, transaction-scoped persistence context, MyBatis 비교, JPA vs MyBatis, 롤백, rollback]
topic: 영속성 컨텍스트가 무엇이고, SQL 을 직접 날리는 방식(MyBatis)과 비교해 무엇을 얻는가
summary: 영속성 컨텍스트는 JPA 스펙이 정의하는 엔티티 저장 공간으로 EntityManager 로 접근하며, 스프링은 그 생명주기를 트랜잭션 범위에 묶는다. 본질은 SQL 이 아니라 엔티티와 스냅샷을 붙잡고 있는 것이고, 그 덕분에 1차 캐시·동일성 보장·변경 감지·쓰기 지연이 나온다. 롤백은 DB 트랜잭션 기능이라 영속성 컨텍스트의 이점이 아니다.
contributors: [dongju]
source_refs:
  - https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html
  - https://jakarta.ee/specifications/persistence/
updated: 2026-10-12
---

# 영속성 컨텍스트 (Persistence Context)

## 개념 정의

JPA 가 엔티티를 관리하는 **저장 공간**. `EntityManager` 를 통해 접근한다.

- **누가 정의하나**: JPA 스펙. 스프링 없이 순수 Java + Hibernate 로도 존재하며, 그때는 `EntityManager` 를 직접 열고 닫는다.
- **스프링은 무엇을 하나**: 트랜잭션이 시작될 때 영속성 컨텍스트를 열고 끝날 때 닫는다 → **트랜잭션 범위 = 영속성 컨텍스트 범위**. (OSIV 가 켜져 있으면 요청 끝까지 늘어난다 → [[open-in-view-osiv]])
- **본질**: SQL 저장소가 아니라 **엔티티(와 스냅샷) 저장소**. SQL 은 그 결과로 만들어지는 부산물이다.

## 작동 원리 — 얻는 것 네 가지

같은 코드를 JPA 와 MyBatis 로 비교하면 차이가 드러난다.

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
| SELECT 횟수 | **1번** | 2번 (호출마다 쿼리) |
| ① `a == b` | **true** | false (결과마다 새 객체) |
| ② update 호출 없이 DB 반영? | **반영됨** | 안 됨 (update 를 직접 호출해야) |

1. **1차 캐시** — 내부에 `@Id → 엔티티` Map. 같은 id 를 다시 조회하면 DB 를 가지 않는다.
2. **동일성 보장** — 같은 영속성 컨텍스트 안에서 같은 id 는 항상 **같은 인스턴스**. `==` 는 참조 비교라 true ([[reference-vs-value-equality]], [[jpa-entity-equality]]).
3. **변경 감지** — 조회 시점의 스냅샷과 현재 값을 플러시 때 비교해 UPDATE 생성 ([[dirty-checking]]).
4. **쓰기 지연** — `persist()`/`remove()` 의 INSERT/DELETE 를 쓰기 지연 SQL 저장소에 모았다가 플러시 때 전송 ([[jpa-flush]]).

### 네 가지는 하나의 뿌리에서 나온다

- "바뀌었다"를 판단하려면 **비교 대상(스냅샷)** 이 필요하다.
- 비교하려면 조회한 엔티티를 반환하고 잊으면 안 되고 **붙잡고 있어야** 한다.
- 그 붙잡고 있는 저장소가 곧 1차 캐시 → 재조회 생략(1차 캐시)과 같은 인스턴스 반환(동일성 보장)이 따라온다.

```
영속성 컨텍스트
 ├─ 1차 캐시        : id → (엔티티, 스냅샷)
 └─ 쓰기 지연 SQL 저장소 : 플러시 때 DB 로 보낼 INSERT / UPDATE / DELETE
```

## 트레이드오프 / 한계

- 영속성 컨텍스트에 머무는 동안 엔티티는 **스냅샷과 함께 두 벌**로 메모리에 있다. 대량 조회·배치에서 부담 → [[dirty-checking]] 의 readOnly, [[jpa-bulk-update-and-clear]] 의 flush → clear.
- 1차 캐시는 **DB 를 직접 고치는 연산(벌크 UPDATE, 네이티브 쿼리)을 모른다** → 캐시가 낡을 수 있다 ([[jpa-bulk-update-and-clear]]).
- 동일성 보장·1차 캐시는 **같은 영속성 컨텍스트 안에서만**. 트랜잭션이 다르면(=컨텍스트가 다르면) 다른 인스턴스이고 SELECT 도 다시 나간다. 요청 간 공유 캐시가 아니다.

> [!WARNING]
> **오답 코너**
> - **"영속성 컨텍스트는 스프링 기능이다"** — JPA 스펙이다. 스프링은 트랜잭션 범위로 생명주기만 관리한다.
> - **"영속성 컨텍스트 덕분에 롤백이 쉽다 / MyBatis 는 롤백이 어렵다"** — 롤백은 **DB 트랜잭션 + `@Transactional`** 의 기능이다. MyBatis 도 중간 예외 시 앞선 INSERT 가 똑같이 롤백된다. 쿼리가 DB 로 "날아간 것"과 "커밋된 것"은 다르다. ([[transactional-rollback-rules]])
> - **"수정·삭제·등록 SQL 을 전부 기록해 뒀다가 커밋 때 반영한다"** — INSERT/DELETE 는 쓰기 지연 저장소에 쌓이지만 **UPDATE 는 미리 기록되지 않고** 플러시 때 스냅샷 비교로 생성된다. 본질은 엔티티 저장소.
> - **"같은 id 두 번 find 하면 SELECT 2번, `a == b` false"** — 1차 캐시로 1번, 동일성 보장으로 true. (MyBatis 의 동작과 혼동)

## 복습 체크

- [ ] 영속성 컨텍스트를 정의하는 주체와, 스프링이 대신 해 주는 일을 구분해 말할 수 있는가?
- [ ] 롤백이 영속성 컨텍스트의 이점이 **아닌** 이유를 MyBatis 예시로 설명할 수 있는가?
- [ ] 같은 id 두 번 `find()` 시 SELECT 횟수와 `==` 결과를 JPA / MyBatis 각각 말할 수 있는가?
- [ ] 1차 캐시·동일성 보장·변경 감지·쓰기 지연 네 가지를 이름과 함께 댈 수 있는가?
- [ ] 네 이점이 "엔티티를 붙잡고 있다"는 하나의 사실에서 나오는 이유를 설명할 수 있는가?
- [ ] 1차 캐시가 요청 간 공유 캐시가 아닌 이유는?

## 관련

[[dirty-checking]] · [[jpa-flush]] · [[jpa-bulk-update-and-clear]] · [[open-in-view-osiv]] · [[jpa-entity-equality]] · [[transactional-rollback-rules]] · [[lazy-loading-proxy]]
