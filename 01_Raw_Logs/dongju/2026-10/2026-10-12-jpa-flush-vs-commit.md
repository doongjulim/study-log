---
date: 2026-10-12
type: raw
author: dongju
tags: [플러시, flush, 커밋, commit, flush vs commit, 자동 플러시, auto flush, FlushModeType, FlushMode.AUTO, JPQL, 쓰기 지연 SQL 저장소, write-behind, 스냅샷, snapshot, 변경 감지, dirty checking, 영속성 컨텍스트, persistence context, 1차 캐시, 롤백, rollback, 동기화, show-sql]
topic: 플러시와 커밋은 다르다 — 플러시는 영속성 컨텍스트 → DB 전송, 커밋은 확정. 자동 플러시는 커밋 직전과 JPQL 실행 직전
summary: 커밋 요청 시 먼저 플러시가 일어나 스냅샷 비교로 UPDATE 를 만들고 쓰기 지연 저장소의 SQL 을 DB 로 보낸 뒤, 마지막에 DB 커밋으로 확정한다. 플러시 후에도 커밋 전이면 롤백 가능하고, 플러시해도 1차 캐시는 비워지지 않는다. JPQL 은 DB 만 보기 때문에 실행 직전에 자동 플러시해 방금 persist 한 엔티티가 조회 결과에서 빠지지 않게 한다.
source: session
distilled: true
distilled_at: 2026-10-12
---

## 배운 개념

### 1. 변경 감지 순서

```java
Member a = em.find(Member.class, 1L);   // (가) 엔티티 + 스냅샷 저장
a.setName("새이름");                     // (나) SQL 안 만들어짐. 자바 필드값만 바뀜
// 트랜잭션 커밋                          // (다)
```

### 2. 커밋 요청 시 일어나는 일

```
트랜잭션 커밋 요청
 → ① 플러시
    → 1차 캐시의 엔티티와 ② 스냅샷 비교
    → 변경된 엔티티의 UPDATE SQL 생성 → ③ 쓰기 지연 SQL 저장소에 등록
    → 쌓인 INSERT/UPDATE/DELETE 를 DB 로 전송   (아직 확정 아님, 롤백 가능)
 → ④ DB 커밋  (여기서 확정)
```

- 1차 캐시(엔티티 담는 칸)와 쓰기 지연 SQL 저장소(SQL 담는 칸)는 둘 다 영속성 컨텍스트 안에 있지만 **서로 다른 칸**.

### 3. 플러시 vs 커밋

> **플러시는 SQL 을 DB 에 보내는 것, 커밋은 보낸 내용을 확정하는 것.**

| 단계 | 택배 비유 | 실제 |
|---|---|---|
| persist / 변경 감지 | 상자 포장해서 창고에 쌓음 | SQL 이 쓰기 지연 저장소에 쌓임 |
| 플러시 | 트럭에 실어 보냄 | SQL 을 DB 로 전송 |
| 커밋 | 받는 사람 수령 서명 | DB 가 변경 확정 |
| 롤백 | 서명 전이면 반송 가능 | 플러시됐어도 커밋 전이면 취소 |

- 방향: 플러시는 **영속성 컨텍스트 → DB** 한 방향.
- 플러시 후 예외 → DB 에 남지 않음 (커밋 전이므로).
- 플러시 후 1차 캐시 → **그대로 남음**. 비우는 건 `clear()`.

### 4. 자동 플러시 시점

1. **트랜잭션 커밋 직전**
2. **JPQL 실행 직전** (기본 `FlushModeType.AUTO`)
3. (수동) `em.flush()` 직접 호출

트랜잭션 **시작** 시점에는 플러시하지 않음 — 영속성 컨텍스트가 비어 있어 동기화할 게 없다.

### 5. 왜 JPQL 직전에 플러시하나

```java
@Transactional
public void test() {
    em.persist(memberA);
    em.createQuery("select m from Member m", Member.class).getResultList();
    throw new RuntimeException();
}
```

```
em.persist(memberA)       → INSERT 가 쓰기 지연 저장소에 쌓임 (DB 엔 아직 안 감)
getResultList()
  → JPQL 직전 자동 플러시  → INSERT 가 DB 로 나감 ★
  → SELECT 실행 (결과에 memberA 포함)
throw RuntimeException    → 트랜잭션 롤백 → 이미 나간 INSERT 도 취소
```

- JPQL 은 SQL 로 변환되어 **DB 만 본다** (1차 캐시를 거치지 않음).
- 그 순간 memberA 는 **영속성 컨텍스트에만** 있고 DB 엔 없다 → 플러시 없이 실행하면 방금 저장한 회원이 결과에서 빠진다.
- 애플리케이션이 아는 상태와 DB 조회 결과의 불일치를 막기 위한 자동 플러시.
- JPA 가 결과 list 에 memberA 를 메모리에서 끼워 넣는 게 **아니다**. list 는 순전히 DB 결과.
- 결론: memberA 는 **나갔지만 커밋되지 않아서** 남지 않음. "안 나가서 안 남는다"와 다르다.

## 오답·헷갈린 점

- **전**: 커밋 순서에서 플러시를 **맨 마지막(④)** 에 둠 → **후**: 플러시가 먼저(①), DB 커밋이 마지막(④). 스스로 "플러시 후 예외 나면 DB 에 안 남는다"고 답한 것과 모순이었음.
- **전**: 쓰기 지연 저장소 자리에 "영속성 컨텍스트에 등록" → **후**: 틀리진 않지만 1차 캐시와 쓰기 지연 SQL 저장소는 다른 칸.
- **전**: persist 직후 JPQL → "포함 안 됨" + "list 에 해당 멤버를 추가함" (자기모순) → **후**: JPQL 직전 **자동 플러시**로 INSERT 가 먼저 DB 로 나가 SELECT 결과에 포함.
- **전**: 자동 플러시 시점 = "트랜잭션의 **시작**과 커밋" → **후**: **커밋 직전**과 **JPQL 실행 직전**.
- **전**: 플러시 = "SQL 쿼리들을 영속성 컨텍스트에 **올리는** 작업" → **후**: 방향 반대. 영속성 컨텍스트 → DB 로 **내보내는** 작업.
- **전**: persist → JPQL → 예외 상황에서 "INSERT 는 DB 로 나가지 않는다, 그래서 안 남는다" → **후**: 자동 플러시로 **나가고**, 롤백으로 취소되어 안 남음. 결론만 맞고 이유가 틀림.
- ⚠️ **플러시의 방향과 시점에서 세 번 흔들림** — 이번 세션 최대 약점. 복습 우선순위 높음.
- ⚠️ 반복 테마: "간극을 메꿔준다" 같은 추상적 인과 표현 → 구체 메커니즘(무엇을 어디로 보내나)으로 환원 필요.

## Q&A

- **Q. 변경 감지 — (가) 조회 시 엔티티 말고 무엇이 저장되나?** A. 스냅샷.
- **Q. (나) setter 시점에 UPDATE SQL 이 생기나?** A. 아니다. 플러시 때 스냅샷 비교로 생성.
- **Q. 플러시 후 RuntimeException → 데이터는?** A. 커밋 전이라 롤백, 남지 않음.
- **Q. 플러시 후 1차 캐시는?** A. 그대로 남음.
- **Q. 자동 플러시 시점 두 가지?** A. 커밋 직전, JPQL 실행 직전.
- **Q. JPQL 직전에 왜 플러시하나?** A. JPQL 은 DB 만 보는데 방금 persist 한 엔티티는 영속성 컨텍스트에만 있어서, 플러시 안 하면 조회 결과에서 빠짐.

## 다음 학습

- **직접 확인**: `spring.jpa.show-sql=true` 로 위 코드 실행 → INSERT 가 SELECT 보다 먼저 찍히는 로그 보기
- `FlushModeType.COMMIT` 과 AUTO 의 차이, Hibernate 의 "쿼리 대상 테이블과 관련 있을 때만 플러시" 최적화
- `em.find()` 는 왜 플러시를 유발하지 않나 (1차 캐시 먼저 조회)
- `@Transactional(readOnly = true)` 와 플러시 모드 관계 → [[dirty-checking]]
