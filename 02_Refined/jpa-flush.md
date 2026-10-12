---
type: refined
slug: jpa-flush
tags: [플러시, flush, em.flush, 커밋, commit, flush vs commit, 플러시와 커밋 차이, 자동 플러시, auto flush, FlushModeType, FlushModeType.AUTO, FlushModeType.COMMIT, FlushMode.MANUAL, JPQL, JPQL 실행 전 플러시, 쓰기 지연 SQL 저장소, write-behind, 쓰기지연, 스냅샷, snapshot, 변경 감지, dirty checking, 영속성 컨텍스트, persistence context, 동기화, synchronization, 롤백, rollback, show-sql]
topic: 플러시는 영속성 컨텍스트 → DB 전송, 커밋은 확정. 자동 플러시가 일어나는 두 시점과 JPQL 직전에 플러시하는 이유
summary: 플러시는 스냅샷 비교로 UPDATE 를 만들고 쓰기 지연 저장소의 SQL 을 DB 로 보내는 동기화 작업이며, 커밋은 보낸 내용을 확정하는 것이다. 플러시 후에도 커밋 전이면 롤백되고, 플러시해도 1차 캐시는 그대로 남는다. 자동 플러시는 커밋 직전과 JPQL 실행 직전에 일어나며, 후자는 DB 만 보는 JPQL 이 방금 persist 한 엔티티를 놓치지 않게 하기 위해서다.
contributors: [dongju]
source_refs:
  - https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html
  - https://jakarta.ee/specifications/persistence/
updated: 2026-10-12
---

# 플러시 (Flush) vs 커밋

## 개념 정의

- **플러시**: 영속성 컨텍스트의 변경 내용을 **DB 로 보내는** 동기화 작업. 방향은 항상 **영속성 컨텍스트 → DB**.
- **커밋**: DB 가 그 변경을 **확정**하는 것.

> 플러시는 SQL 을 DB 에 **보내는** 것, 커밋은 보낸 내용을 **확정**하는 것.

| 단계 | 택배 비유 | 실제 |
|---|---|---|
| persist / 변경 감지 | 상자를 포장해 창고에 쌓음 | SQL 이 쓰기 지연 저장소에 쌓임 |
| 플러시 | 트럭에 실어 보냄 | SQL 을 DB 로 전송 |
| 커밋 | 받는 사람 수령 서명 | DB 가 변경 확정 |
| 롤백 | 서명 전이면 반송 | 플러시됐어도 커밋 전이면 취소 |

## 작동 원리

### 커밋 요청 시 순서

```
트랜잭션 커밋 요청
 → ① 플러시
    → 1차 캐시의 엔티티와 ② 스냅샷 비교
    → 변경된 엔티티의 UPDATE SQL 생성 → ③ 쓰기 지연 SQL 저장소에 등록
    → 쌓인 INSERT / UPDATE / DELETE 를 DB 로 전송   (아직 확정 아님)
 → ④ DB 커밋   (여기서 확정)
```

- 1차 캐시(엔티티 칸)와 쓰기 지연 SQL 저장소(SQL 칸)는 둘 다 영속성 컨텍스트 안이지만 **다른 칸**이다.
- 플러시 후 **1차 캐시는 그대로** 남는다. 비우는 건 `em.clear()` ([[jpa-bulk-update-and-clear]]).

### 자동 플러시 시점 (기본 `FlushModeType.AUTO`)

1. **트랜잭션 커밋 직전**
2. **JPQL 실행 직전**
3. (수동) `em.flush()` 직접 호출

트랜잭션 **시작** 시점에는 플러시할 게 없다 — 컨텍스트가 비어 있다.

### 왜 JPQL 직전에 플러시하나

```java
@Transactional
public void test() {
    em.persist(memberA);
    em.createQuery("select m from Member m", Member.class).getResultList();
    throw new RuntimeException();
}
```

```
em.persist(memberA)       → INSERT 가 쓰기 지연 저장소에 쌓임 (DB 엔 아직 없음)
getResultList()
  → JPQL 직전 자동 플러시  → INSERT 가 DB 로 나감 ★
  → SELECT 실행 (결과에 memberA 포함)
throw RuntimeException    → 롤백 → 이미 나간 INSERT 도 취소
```

- JPQL 은 SQL 로 변환되어 **DB 만 본다**(1차 캐시를 안 거침).
- 그 순간 memberA 는 **영속성 컨텍스트에만** 있다 → 플러시 없이 SELECT 하면 방금 저장한 회원이 결과에서 빠진다.
- 애플리케이션이 아는 상태와 DB 조회 결과의 불일치를 막기 위한 자동 플러시다.
- 결론: memberA 는 **나갔지만 커밋되지 않아서** 남지 않는다.

`spring.jpa.show-sql=true` 로 실행하면 INSERT 가 SELECT 보다 **먼저** 찍히는 걸 직접 볼 수 있다.

## 트레이드오프 / 한계

- JPQL 마다 자동 플러시가 일어나므로, 쓰기 지연으로 모아 두려던 SQL 이 중간에 흩어져 나갈 수 있다. (Hibernate 는 쿼리 대상 테이블과 관련 있을 때만 플러시하는 최적화를 한다.)
- `FlushModeType.COMMIT` 으로 바꾸면 커밋 때만 플러시 → JPQL 결과가 컨텍스트 상태와 어긋날 수 있다.
- `@Transactional(readOnly = true)` 에서는 플러시 모드가 `MANUAL` 이 되어 커밋 때도 플러시하지 않는다 ([[dirty-checking]]).
- 벌크 연산도 JPQL 이라 **직전** 플러시는 되지만, **이후** 1차 캐시는 갱신되지 않는다 ([[jpa-bulk-update-and-clear]]).

> [!WARNING]
> **오답 코너**
> - **커밋 순서에서 플러시를 맨 마지막에 둠** — 플러시가 먼저, DB 커밋이 마지막. "플러시 후 예외 나면 DB 에 안 남는다"가 맞다면 플러시는 아직 트랜잭션 안이다.
> - **"플러시 = SQL 을 영속성 컨텍스트에 올리는 작업"** — 방향이 반대. 쌓는 건 persist/변경 감지, 플러시는 DB 로 **내보낸다**.
> - **"자동 플러시는 트랜잭션 시작과 커밋 때"** — 시작이 아니라 **커밋 직전과 JPQL 실행 직전**.
> - **"persist 직후 JPQL 의 결과에 memberA 는 없다 / 있다면 JPA 가 list 에 끼워 넣은 것"** — JPQL 직전 자동 플러시로 INSERT 가 먼저 DB 로 나가서 SELECT 결과에 포함된다. list 는 순전히 DB 결과다.
> - **"그 INSERT 는 DB 로 안 나가서 안 남는다"** — 나가지만 **커밋되지 않아서** 안 남는다. 결론만 같고 이유가 다르면 꼬리 질문에 무너진다.
> - **"간극을 메꿔준다"** 처럼 추상적으로 끝내지 말 것 — "영속성 컨텍스트에만 있는 변경을 JPQL 전에 DB 로 보내 조회 결과에 포함되게 한다".

## 복습 체크

- [ ] 플러시와 커밋의 차이를 한 문장으로 말할 수 있는가?
- [ ] 커밋 요청 시 순서를 플러시·스냅샷 비교·쓰기 지연 저장소·DB 커밋으로 말할 수 있는가?
- [ ] 플러시 후 예외가 나면 데이터는? 플러시 후 1차 캐시는?
- [ ] 자동 플러시 시점 두 가지는? 트랜잭션 시작 때 플러시하지 않는 이유는?
- [ ] persist → JPQL → 예외 코드에서 INSERT 가 언제 나가고 최종적으로 남는지 설명할 수 있는가?
- [ ] JPQL 직전에 왜 플러시하는가?

## 관련

[[persistence-context]] · [[dirty-checking]] · [[jpa-bulk-update-and-clear]] · [[transactional-rollback-rules]] · [[open-in-view-osiv]]
