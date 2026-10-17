---
type: refined
slug: dirty-checking
tags: [변경감지, dirty-checking, 스냅샷, snapshot, flush, 플러시, 쓰기지연, write-behind, 1차캐시, first-level-cache, 영속성컨텍스트, persistence-context, 영속, managed, 준영속, detached, merge, readOnly, transactional-readonly, FlushMode.MANUAL, 읽기 전용 세션, read-only session, 전체 컬럼 업데이트, all columns update, DynamicUpdate, "@DynamicUpdate", PreparedStatement, 실행 계획 캐시, JDBC 배치, JDBC batch, 변경 감지 비용, dirty checking cost, 전체 엔티티 비교, JPA, Hibernate]
topic: 영속 엔티티의 값 변경이 UPDATE 로 바뀌는 시점과 그 판정 기준(스냅샷 비교)
summary: 영속성 컨텍스트는 엔티티를 읽을 때 스냅샷을 떠두고, flush 시점에 현재 값과 비교해 달라진 것만 UPDATE 로 만든다. 값을 바꾸는 순간이 아니라 flush 가 트리거이며, 준영속 엔티티는 비교 대상이 아니다. setter 는 아무 기록도 남기지 않아 flush 때마다 모든 영속 엔티티를 필드 단위로 비교하므로 메모리(스냅샷)와 CPU 비용이 들고, 조회 전용은 readOnly = true 로 둘 다 생략한다. 기본 UPDATE 는 SQL 재사용을 위해 전체 컬럼을 SET 한다.
contributors: [dongju]
source_refs:
  - https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html
  - https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html
updated: 2026-10-17
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
플러시 자체(커밋과의 차이, JPQL 직전 자동 플러시의 이유)는 [[jpa-flush]] 참고.

### 실제로 나가는 UPDATE — 전체 컬럼

```java
Member m = em.find(Member.class, id);   // 컬럼: id, name, age, email, address ...
m.setName("새이름");
```

```sql
update member set name=?, age=?, email=?, address=?, ... where id=?
```

`name` 하나만 바꿔도 **id 를 뺀 모든 컬럼**을 SET 한다 (id 는 WHERE 의 키). 이유:

1. **Hibernate 쪽** — 모양이 엔티티당 하나라 **애플리케이션 시작(부트) 시** INSERT/UPDATE/DELETE SQL 을 미리 만들어 캐시해 둔다. 바뀐 컬럼만 보내면 컬럼 10개 기준 조합이 2¹⁰ − 1 = **1023가지**라 미리 못 만들고 flush 마다 SQL 을 새로 생성해야 한다.
2. **DB 쪽** — 글자 하나 다르지 않은 같은 SQL(`?` 자리표시자, `PreparedStatement`)이면 파싱 결과·실행 계획 캐시를 재사용한다.
3. **JDBC 배치** — 여러 UPDATE 를 한 번에 묶으려면 SQL 모양이 같아야 한다 (`hibernate.jdbc.batch_size`, [[jpa-bulk-update-and-clear]]).

바뀐 컬럼만 보내려면 엔티티에 **`@DynamicUpdate`**. 컬럼이 아주 많거나 큰 컬럼(TEXT, BLOB)이 있어 전송량 절감 이득이 SQL 재생성 비용보다 클 때 쓴다.

### 상태에 따른 차이

| 상태 | 스냅샷 비교 대상? | 값 변경 시 |
|---|---|---|
| 영속 (managed) | O | flush 때 UPDATE |
| 준영속 (detached) | X | 아무 일도 안 일어남 |
| 비영속 (transient) | X | 아무 일도 안 일어남 |

트랜잭션이 끝나며 영속성 컨텍스트가 닫히면 엔티티는 **준영속**이 된다. 준영속 엔티티를 다시
관리 대상으로 만들려면 `merge()` 를 쓰거나, 다시 조회해서 값을 옮긴다.

> **정정 (2026-10-17)** — 이전 표현 "merge = 복사본을 새로 영속화"는 부정확했다. `merge()` 는 인자를 영속화하지 않는다.
> 인자의 id 로 1차 캐시나 DB 에서 **영속 엔티티를 얻고**, 거기에 인자의 **모든 필드를 덮어쓴 뒤** 그 영속 객체를 반환한다.
> 인자는 준영속 그대로다. 폼에 없던 필드까지 null·0 으로 덮이므로, 수정은 **조회 + 엔티티 메서드 + 변경 감지**가 안전하다 ([[jpa-entity-lifecycle-merge]]).

## 트레이드오프 / 한계

- **스냅샷은 메모리 비용이다.** 엔티티마다 복사본이 하나씩 더 생기므로 대량 조회 시 메모리 부담이 대략 두 배
  (조회 1만 건 = 엔티티 1만 + 스냅샷 1만).
- **비교는 CPU 비용이다.** `setName()` 은 순수 필드 대입이라 Hibernate 에 **아무 기록도 남기지 않는다**.
  "바뀐 것만 고른다"는 판단 자체가 비교의 결과이므로, flush 때마다 **모든 영속 엔티티를 필드 단위로** 비교한다.

  ```
  조회만 하는 메서드, 1만 건 × 필드 10개
   → 커밋 직전 flush: 비교 약 100,000번 → 바뀐 것 0건, UPDATE 0개
  ```

  (바이트코드 강화로 setter 가 변경을 직접 기록하게 할 수 있으나 기본값은 아니다.)
- → `@Transactional(readOnly = true)` 는 관례가 아니라 **최적화**다. 스프링 + Hibernate 에서
  ① 세션이 읽기 전용이 되어 **스냅샷을 만들지 않고**(스프링 5.1+) ② 플러시 모드가 **`MANUAL`** 이 되어 커밋 때 **flush 를 생략**한다.
  덤으로 JDBC 커넥션에 읽기 전용 힌트가 전달돼 복제본 라우팅 등에 쓰인다. 조회 전용 메서드에는 반드시 붙일 것. ([[db-pagination]])
- **readOnly 안에서 수정하면 예외 없이 조용히 무시된다.** 스냅샷이 없어 비교 불가 + flush 자체가 없음 → 이중으로 막힌다.
  "저장했는데 왜 안 바뀌지?" 버그를 막기 위해 흔히 이렇게 쓴다:

  ```java
  @Service
  @Transactional(readOnly = true)      // 클래스 기본값: 조회 전용
  public class MemberService {
      public MemberDto find(Long id) { ... }

      @Transactional                   // 쓰기 메서드만 덮어쓰기
      public void changeName(Long id, String name) { ... }
  }
  ```
- **flush 가 컨텍스트 전체를 훑는다는 점이 위험하다.** 특정 엔티티만 골라 반영하는 게 아니라
  컨텍스트 안의 모든 영속 엔티티가 비교 대상이다. OSIV 가 켜져 있으면 컨트롤러에서 바꾼 값이
  나중에 열린 **무관한 트랜잭션의 커밋에 딸려 나간다** ([[open-in-view-osiv]]).
- **부분 업데이트가 안 된다** — 기본 설정에서 Hibernate 는 변경된 컬럼만이 아니라
  **엔티티의 모든 컬럼**을 UPDATE 문에 넣는다. 비효율처럼 보이지만 SQL 재사용·실행 계획 캐시·JDBC 배치를 위한 선택이다
  (위 "실제로 나가는 UPDATE" 참고, `@DynamicUpdate` 로 변경 가능).

> [!WARNING]
> **오답 코너**
> - **"값을 바꾸면 즉시 UPDATE 가 나간다"** — 아니다. **flush 시점**에 나간다. 이 시차가 핵심.
> - **스냅샷의 이름·정체를 "데모 객체" 등으로 잘못 기억** — 조회 시점 상태의 복사본이며,
>   flush 때 현재 엔티티와 비교되어 UPDATE 를 만든다.
> - **"트랜잭션 밖에서 바꾸면 절대 반영 안 된다"** — 엔티티가 **여전히 영속 상태라면 반영될 수 있다**
>   (OSIV 켜짐 + 후속 트랜잭션). 판단 기준은 "트랜잭션 안/밖"이 아니라 **"영속인가 준영속인가"** 다.
> - **"`readOnly = true` 는 그냥 의도 표시"** — 스냅샷 생략 + flush 생략이라는 실제 최적화가 걸린다.
> - **"flush 때 수정된 엔티티만 골라 비교한다"** — setter 는 기록을 안 남긴다. 바뀐 걸 알려면 **모든 영속 엔티티**를 비교해야 한다. 조회만 1만 건이어도 전부 비교.
> - **"조회 1만 건이면 메모리에 스냅샷 1만 개"** — **엔티티 1만 + 스냅샷 1만**, 두 벌이다.
> - **"비교 시점은 트랜잭션 종료 시"** — 정확히는 **flush** 때이고, flush 는 커밋 직전 **그리고 JPQL 직전**에도 일어난다. (2026-10 재학습에서도 JPQL 쪽을 빠뜨림)
> - **"전체 컬럼 UPDATE 는 그냥 더 합리적이라서"** — 추상적 답변. 구체적으로 ① 부트 시 SQL 미리 생성 ② DB 실행 계획 캐시 ③ JDBC 배치. SQL 생성 시점은 컴파일이 아니라 **애플리케이션 시작** 시.
> - **"readOnly 안에서 수정하면 예외가 난다"** — 안 난다. 조용히 무시된다.

## 복습 체크

- [ ] 스냅샷이 언제 만들어지고 어디에 보관되는지 말할 수 있는가?
- [ ] UPDATE 가 나가는 트리거가 "값 변경"이 아니라 "flush" 임을 설명할 수 있는가?
- [ ] flush 가 자동으로 일어나는 두 시점을 댈 수 있는가?
- [ ] 영속/준영속/비영속 중 어느 상태가 변경 감지 대상인가?
- [ ] `@Transactional(readOnly = true)` 가 실제로 무엇을 생략하는가?
- [ ] flush 가 "컨텍스트 전체"를 훑는다는 성질이 어떤 사고로 이어지는가?
- [ ] 수정이 없는 조회 전용 메서드에서도 비교 비용이 드는 이유를 setter 와 연결해 설명할 수 있는가?
- [ ] `name` 만 바꿨을 때 나가는 UPDATE SQL 을 쓰고, 전체 컬럼인 이유 세 가지를 댈 수 있는가?
- [ ] `@DynamicUpdate` 는 언제 쓰는가?
- [ ] readOnly 메서드 안에서 엔티티를 수정하면 어떻게 되는가? 이유 두 가지는?

## 관련

[[jpa-entity-lifecycle-merge]] · [[persistence-context]] · [[jpa-flush]] · [[jpa-bulk-update-and-clear]] · [[open-in-view-osiv]] · [[lazy-loading-proxy]] · [[transactional-event-listener]] · [[db-pagination]]
