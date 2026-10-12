---
date: 2026-10-12
type: raw
author: dongju
tags: [변경 감지 비용, dirty checking cost, 더티 체킹 비용, 스냅샷 메모리, snapshot memory, 전체 엔티티 비교, full comparison, 필드 단위 비교, readOnly, Transactional readOnly, 읽기 전용 트랜잭션, read-only transaction, FlushMode.MANUAL, 플러시 모드, 읽기 전용 세션, read-only session, 조회 전용, 바이트코드 강화, bytecode enhancement, 성능, performance, 클래스 레벨 Transactional]
topic: 변경 감지의 비용(메모리 두 벌, 플러시마다 전체 엔티티 필드 비교)과 @Transactional(readOnly = true) 로 조회 전용 메서드의 비용을 없애는 방법
summary: setter 는 Hibernate 에 아무 기록도 남기지 않으므로, 플러시 때 Hibernate 는 수정 여부와 상관없이 모든 영속 엔티티를 스냅샷과 필드 단위로 비교해야 바뀐 걸 안다. 조회만 1만 건이면 엔티티+스냅샷 두 벌이 메모리에 올라가고 약 10만 번 비교 후 UPDATE 0개다. readOnly = true 는 스냅샷 생성 생략 + 플러시 모드 MANUAL 로 이 비용을 없애며, 그 안에서 수정하면 예외 없이 조용히 무시된다.
source: session
distilled: true
distilled_at: 2026-10-12
---

## 배운 개념

### 1. 플러시 때 비교 대상은 "전부"

```java
@Transactional
public List<MemberDto> findAll() {
    List<Member> members = em.createQuery("select m from Member m", Member.class)
                             .getResultList();   // 1만 건
    return members.stream().map(MemberDto::from).toList();
}   // 커밋 → 플러시
```

- `setName()` 은 `this.name = ...` 일 뿐, Hibernate 코드는 한 줄도 안 돎 → 어떤 엔티티가 바뀌었는지 **기록이 없다**.
- "바뀐 것만 고른다"는 판단 자체가 비교의 **결과** → 비교 대상을 미리 좁힐 수 없음.

```
findAll() 커밋 직전 플러시
 → 영속 엔티티 10,000건 전부 순회
    → 엔티티마다 필드 10개를 스냅샷과 하나씩 비교
 → 비교 약 100,000번, 결과: 바뀐 것 0건, UPDATE 0개
```

### 2. 비용

| 비용 | 원인 |
|---|---|
| **메모리** | 엔티티마다 스냅샷 한 벌 → **엔티티 1만 + 스냅샷 1만**, 같은 데이터 두 벌 |
| **CPU** | 플러시마다 **모든** 영속 엔티티를 필드 단위 비교. 수정이 없어도 비교 |

- 가장 아까운 곳: **조회 전용 메서드**. 안 바꿀 건데 스냅샷 뜨고 10만 번 비교해서 "변경 없음" 확인.
- 참고: 바이트코드 강화(bytecode enhancement)로 setter 호출 시 변경을 기록하게 할 수 있으나 기본값 아님. 면접에서는 "기본 동작은 전체 비교".

### 3. 해결 — `@Transactional(readOnly = true)`

스프링 + Hibernate 에서 바뀌는 것:

1. 플러시 모드 **`MANUAL`** → 커밋 때 자동 플러시 없음 → 스냅샷 비교 없음
2. Hibernate 세션 **읽기 전용** → 조회 시 **스냅샷을 만들지 않음** (스프링 5.1+)
3. 덤: JDBC 커넥션에 읽기 전용 힌트 → DB/라우팅 설정에 따라 읽기 복제본(replica)으로 보내는 데 활용

### 4. readOnly 안에서 수정하면 — 조용히 무시

```java
@Transactional(readOnly = true)
public void oops(Long id) {
    Member m = em.find(Member.class, id);
    m.setName("새이름");   // DB 안 바뀜
}
```

이중으로 막힘:

- 스냅샷이 없음 → 비교 대상 없음 → 변경 감지 불가
- 플러시 모드 `MANUAL` → 커밋 때 플러시 자체가 없음 → SQL 이 나갈 기회 없음

⚠️ **예외가 안 난다** → "저장했는데 왜 안 바뀌지?" 버그.

### 5. 실무 패턴

```java
@Service
@Transactional(readOnly = true)      // 클래스 기본값: 조회 전용
public class MemberService {

    public MemberDto find(Long id) { ... }

    @Transactional                   // 쓰기 메서드만 덮어쓰기
    public void changeName(Long id, String name) { ... }
}
```

### 6. 면접 답변 뼈대

> "변경 감지는 공짜가 아닙니다. 엔티티마다 스냅샷을 한 벌 더 들고 있어서 메모리가 두 배로 들고, setter 는 Hibernate 에 아무 기록도 남기지 않기 때문에 플러시 때마다 모든 영속 엔티티를 필드 단위로 비교해야 합니다. 조회만 하는 메서드에서도 이 비용이 그대로 들기 때문에, 서비스 클래스에 `@Transactional(readOnly = true)` 를 기본으로 두고 쓰기 메서드만 `@Transactional` 로 덮어씁니다. readOnly 면 스냅샷을 만들지 않고 플러시도 하지 않습니다. 대신 그 안에서 엔티티를 수정하면 예외 없이 무시되니 주의해야 합니다."

## 오답·헷갈린 점

- **전**: 플러시 때 "**수정된 엔티티만** 골라서 비교" → **후**: setter 는 기록을 안 남기므로 **모든 영속 엔티티**를 비교해야 바뀐 걸 앎. 힌트 후 재질문에도 "비교해서 변경 있으면 전체 컬럼 업데이트"로 동작만 되풀이하고 **비교 건수**는 답하지 못함 → 설명으로 정리.
- **전**: 조회 1만 건 시 메모리에 "**스냅샷 1만 개**" → **후**: **엔티티 1만 + 스냅샷 1만**, 두 벌.
- ✅ 해결책 `readOnly = true` 즉답.
- ✅ readOnly 안의 수정이 반영 안 되는 이유를 스냅샷과 연결해 설명. (플러시 모드 MANUAL 쪽 이유는 보충으로 정리)
- ⚠️ 이번 세션에서 질문의 **일부만 답하는** 패턴 반복(플러시 시점 절반, 비교 건수 미답). 면접에서는 질문을 끝까지 듣고 모든 하위 질문에 답하기.

## Q&A

- **Q. 플러시 때 수정된 엔티티만 비교하나?** A. 아니다. 기록이 없으니 모든 영속 엔티티를 비교.
- **Q. 조회만 1만 건, 필드 10개 → 비교 횟수와 UPDATE 수?** A. 약 10만 번, UPDATE 0개.
- **Q. 메모리에 올라가는 것?** A. 엔티티 1만 + 스냅샷 1만.
- **Q. 조회 전용 메서드의 비용 제거?** A. `@Transactional(readOnly = true)` → 스냅샷 생략 + 플러시 MANUAL.
- **Q. readOnly 안에서 setName → DB 반영?** A. 안 됨. 스냅샷 없음 + 플러시 없음. 예외도 안 남.

## 다음 학습

- readOnly 트랜잭션과 읽기 복제본 라우팅 (`AbstractRoutingDataSource`, `LazyConnectionDataSourceProxy`)
- 대량 조회는 엔티티 대신 **DTO 프로젝션**으로 → 영속성 컨텍스트에 안 올라가서 스냅샷·비교 비용 0
- Hibernate 바이트코드 강화의 dirty tracking 동작
- 앞선 노트: [[2026-10-12-dirty-checking-full-column-update]], [[2026-10-12-jpa-persistence-context-benefits]]
