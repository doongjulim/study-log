---
type: refined
slug: jpa-specification-dynamic-query
tags: [Specification, JpaSpecificationExecutor, Criteria, CriteriaBuilder, CriteriaAPI, 동적쿼리, 동적 쿼리, dynamic-query, 선택조건, optional-condition, 조합폭발, 2의n승, is-null-or, 조건조립, 필드이름문자열, root.get, 런타임에러, IllegalArgumentException, 정적메타모델, static-metamodel, jpamodelgen, Querydsl, BooleanBuilder, BooleanExpression, 컴파일타임검사, 객체레벨, JPA, Spring-Data-JPA, 면접답변]
topic: 선택 조건이 여러 개 자유롭게 조합되면 Specification 으로 조건을 조각내 이어 붙인다(n 개 → 2ⁿ 조합). 조건이 하나면 JPQL 의 is null or 로 충분하다. 약점은 필드 이름이 문자열이라 컴파일 때 못 잡는 것
summary: 있을 수도 없을 수도 있는 조건이 n 개면 조합은 2ⁿ 가지라, 조합마다 JPQL 을 만들거나 :param is null or 를 n 줄 늘어놓으면 읽기 어렵고 파라미터 타입 문제도 생긴다. Specification 은 조건 하나를 메서드 하나로 떼어 있을 때만 and() 로 이어 붙인다. 선택 조건이 하나뿐이면 JPQL + is null or 한 줄로 충분하다. Criteria 는 JPQL 을 자바로 조립하는 객체 레벨 API 라 root.get("title") 의 "title" 은 컬럼이 아니라 엔티티 필드 이름 문자열이고, 이름이 바뀌어도 컴파일러가 못 잡아 실행 시 에러가 난다 → 정적 메타모델이나 Querydsl 로 컴파일 시점 검사.
contributors: [dongju]
source_refs:
  - https://docs.spring.io/spring-data/jpa/reference/jpa/specifications.html
  - https://jakarta.ee/specifications/persistence/3.1/
updated: 2026-10-18
---

# Specification 과 동적 쿼리

## 개념 정의

**동적 쿼리** = 입력에 따라 where 조건이 붙었다 빠졌다 하는 쿼리. 검색 화면의 선택 조건이 대표적이다.

조건이 n 개면 각각 "있다 / 없다" 2가지라 조합은 **2ⁿ**.

```
키워드 × 카테고리 × 완료 여부 × 시작일 × 종료일
  2    ×    2     ×    2     ×   2   ×   2    = 2⁵ = 32
```

- 조합마다 JPQL → 메서드 32개.
- `:param is null or ...` 로 몰아넣기 → null 체크 5줄 + 파라미터 타입 추론 문제. 조건이 하나 늘 때마다 두 배.

## 작동 원리

```java
public final class PlanSpecifications {
    public static Specification<Plan> ownedBy(Long memberId) {          // 항상 거는 조각
        return (root, query, cb) -> cb.equal(root.get("author").get("id"), memberId);
    }
    public static Specification<Plan> matching(PlanSearchCondition c) { // 있을 때만 이어 붙임
        Specification<Plan> spec = (root, query, cb) -> cb.conjunction();
        if (c.hasKeyword())  spec = spec.and(titleOrContentContains(c.keyword()));
        if (c.hasCategory()) spec = spec.and((root, q, cb) -> cb.equal(root.get("category"), c.category()));
        if (c.from() != null) spec = spec.and((root, q, cb) -> cb.greaterThanOrEqualTo(root.get("planDate"), c.from()));
        ...
        return spec;
    }
}

// Repository extends JpaSpecificationExecutor<Plan>
planRepository.findAll(ownedBy(memberId).and(matching(condition)), pageable);
```

- **조건 하나 = 메서드 하나** → 조건을 더할 때 이 파일만 연다.
- Specification 은 Criteria API 를 감싼 것으로, 결국 JPQL 수준의 쿼리를 **자바 코드로 조립**한다.

### 선택 조건이 하나면 JPQL 로 충분

```java
findSummariesByTitle(keyword, category, pageable)          // 검색 종류는 메서드로 분리
findSummariesByAuthorNickname(keyword, category, pageable)
// + " and (:category is null or p.category = :category)"   // 선택 조건은 카테고리 하나
```

> **기준**: 선택 조건이 한두 개 → JPQL + `is null or`. 여러 개가 자유롭게 조합 → Specification(또는 Querydsl).

## 트레이드오프 / 한계 — 필드 이름이 문자열

```java
cb.like(cb.lower(root.get("title")), pattern)
```

- Criteria 는 **객체 레벨** → `"title"` 은 **엔티티 필드 이름** (컬럼이 `plan_title` 이어도 `"title"`). DB 레벨은 Native 뿐.
- 그냥 `String` 이라 컴파일러가 검사하지 못한다 → 필드 이름을 바꾸고 놓치면 **실행해서 검색할 때** `IllegalArgumentException`.

| 방법 | 코드 | 필드 이름이 바뀌면 |
|---|---|---|
| 문자열 | `root.get("title")` | **실행 시** 에러 |
| JPA 정적 메타모델 | `root.get(Plan_.title)` | **컴파일** 에러 (`Plan_` 은 빌드 때 생성) |
| Querydsl | `QPlan.plan.title.containsIgnoreCase(kw)` | **컴파일** 에러 + 가독성 |

- 그 밖에: Criteria 코드는 장황하고 읽기 어렵다. 동적 쿼리가 많아지면 Querydsl 이 흔한 선택이다.

> **면접 답변** — "Specification은 선택 조건이 여러 개 자유롭게 조합될 때 씁니다. 계획 검색은 조건이 다섯 개라 조합이 32가지여서, 조건 하나를 메서드 하나로 떼고 있을 때만 and로 이어 붙였습니다. 게시글 검색처럼 선택 조건이 하나뿐이면 JPQL로 충분합니다. 약점은 root.get("title")처럼 필드 이름이 문자열이라 이름이 바뀌어도 컴파일 때 잡히지 않는다는 점이고, 동적 쿼리가 늘면 정적 메타모델이나 Querydsl로 옮기겠습니다."

> [!WARNING]
> **오답 코너**
> - **"선택 조건 5개의 조합은 25"** — 있다/없다 2가지씩 독립 → **2⁵ = 32**.
> - **"게시글 검색의 선택 조건은 5개"** — 검색 종류는 메서드로 분리했고 선택 조건은 **카테고리 하나**.
> - **"root.get("title") 의 title 은 DB 컬럼 이름"** — **엔티티 필드 이름**. Criteria 는 객체 레벨.

## 복습 체크

- [ ] 선택 조건이 n 개일 때 조합 수와, JPQL 로 버티기 어려운 이유는?
- [ ] Specification 으로 조건을 조립하는 방식을 설명할 수 있는가?
- [ ] JPQL 의 `is null or` 로 충분한 경우와 Specification 이 필요한 경우를 가를 수 있는가?
- [ ] `root.get("title")` 의 "title" 은 무엇이며, 이름이 바뀌면 언제 에러가 나는가?
- [ ] 컴파일 시점에 잡는 대안 두 가지는?

## 관련

[[jpa-jpql-vs-native-query]] · [[derived-query-method]] · [[like-search-btree-index]] · [[db-pagination]]
