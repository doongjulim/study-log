---
type: refined
slug: dirty-checking
tags: [변경감지, dirty-checking, 스냅샷, snapshot, flush, 플러시, 쓰기지연, write-behind, 1차캐시, first-level-cache, 영속성컨텍스트, persistence-context, 영속, managed, 준영속, detached, merge, readOnly, transactional-readonly, JPA, Hibernate]
topic: 영속 엔티티의 값 변경이 UPDATE 로 바뀌는 시점과 그 판정 기준(스냅샷 비교)
summary: 영속성 컨텍스트는 엔티티를 읽을 때 스냅샷을 떠두고, flush 시점에 현재 값과 비교해 달라진 것만 UPDATE 로 만든다. 값을 바꾸는 순간이 아니라 flush 가 트리거이며, 준영속 엔티티는 비교 대상이 아니다.
contributors: [dongju]
source_refs:
  - https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html
updated: 2026-08-25
---

# 변경 감지 (Dirty Checking)

## 개념 정의

영속 상태(managed) 엔티티의 필드를 바꾸면 `save()` 를 호출하지 않아도 `UPDATE` 가 나가는 JPA 의 동작.
근거는 **스냅샷 비교**다.

- **스냅샷**: 영속성 컨텍스트가 엔티티를 **처음 읽어올 때 떠두는 그 시점 상태의 복사본**.
  1차 캐시에 `(식별자 → 엔티티, 스냅샷)` 형태로 함께 보관된다.
- **flush**: 컨텍스트의 변경분을 DB 에 반영하는 동기화 작업. **여기서 스냅샷과 현재 값을 비교**한다.

## 작동 원리

```
① 조회 → 1차 캐시에 엔티티 + 스냅샷 저장
② post.setTitle("변경")     ← 이 시점에는 아무 쿼리도 나가지 않는다
③ flush                     ← 스냅샷과 현재 값 비교 → 달라진 엔티티만 UPDATE SQL 생성
④ 쓰기 지연 SQL 저장소의 SQL 을 DB 로 전송
⑤ commit
```

**flush 가 일어나는 시점**

1. 트랜잭션 **커밋 직전** (자동)
2. **JPQL/쿼리 실행 직전** (자동 — 아직 반영 안 된 변경분이 쿼리 결과에 빠지는 것을 막기 위해)
3. `em.flush()` 직접 호출

즉 **"값을 바꾼 순간"이 아니라 "flush 순간"이 트리거**다. 이 시차가 여러 함정의 뿌리다.

### 상태에 따른 차이

| 상태 | 스냅샷 비교 대상? | 값 변경 시 |
|---|---|---|
| 영속 (managed) | O | flush 때 UPDATE |
| 준영속 (detached) | X | 아무 일도 안 일어남 |
| 비영속 (transient) | X | 아무 일도 안 일어남 |

트랜잭션이 끝나며 영속성 컨텍스트가 닫히면 엔티티는 **준영속**이 된다. 준영속 엔티티를 다시
관리 대상으로 만들려면 `merge()`(복사본을 새로 영속화)를 쓰거나, 다시 조회해서 값을 옮긴다.

## 트레이드오프 / 한계

- **스냅샷은 메모리 비용이다.** 엔티티마다 복사본이 하나씩 더 생기므로 대량 조회 시 메모리 부담이 대략 두 배.
  → `@Transactional(readOnly = true)` 는 관례가 아니라 **최적화**다. Hibernate 가
  **스냅샷 생성과 flush 를 생략**한다. 조회 전용 메서드에는 반드시 붙일 것. ([[db-pagination]])
- **flush 가 컨텍스트 전체를 훑는다는 점이 위험하다.** 특정 엔티티만 골라 반영하는 게 아니라
  컨텍스트 안의 모든 영속 엔티티가 비교 대상이다. OSIV 가 켜져 있으면 컨트롤러에서 바꾼 값이
  나중에 열린 **무관한 트랜잭션의 커밋에 딸려 나간다** ([[open-in-view-osiv]]).
- **부분 업데이트가 안 된다** — 기본 설정에서 Hibernate 는 변경된 컬럼만이 아니라
  **엔티티의 모든 컬럼**을 UPDATE 문에 넣는다(`@DynamicUpdate` 로 변경 가능).

> [!WARNING]
> **오답 코너**
> - **"값을 바꾸면 즉시 UPDATE 가 나간다"** — 아니다. **flush 시점**에 나간다. 이 시차가 핵심.
> - **스냅샷의 이름·정체를 "데모 객체" 등으로 잘못 기억** — 조회 시점 상태의 복사본이며,
>   flush 때 현재 엔티티와 비교되어 UPDATE 를 만든다.
> - **"트랜잭션 밖에서 바꾸면 절대 반영 안 된다"** — 엔티티가 **여전히 영속 상태라면 반영될 수 있다**
>   (OSIV 켜짐 + 후속 트랜잭션). 판단 기준은 "트랜잭션 안/밖"이 아니라 **"영속인가 준영속인가"** 다.
> - **"`readOnly = true` 는 그냥 의도 표시"** — 스냅샷 생략 + flush 생략이라는 실제 최적화가 걸린다.

## 복습 체크

- [ ] 스냅샷이 언제 만들어지고 어디에 보관되는지 말할 수 있는가?
- [ ] UPDATE 가 나가는 트리거가 "값 변경"이 아니라 "flush" 임을 설명할 수 있는가?
- [ ] flush 가 자동으로 일어나는 두 시점을 댈 수 있는가?
- [ ] 영속/준영속/비영속 중 어느 상태가 변경 감지 대상인가?
- [ ] `@Transactional(readOnly = true)` 가 실제로 무엇을 생략하는가?
- [ ] flush 가 "컨텍스트 전체"를 훑는다는 성질이 어떤 사고로 이어지는가?

## 관련

[[open-in-view-osiv]] · [[lazy-loading-proxy]] · [[transactional-event-listener]] · [[db-pagination]]
