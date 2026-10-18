---
date: 2026-10-18
type: raw
author: dongju
tags: [Specification, Criteria API, CriteriaBuilder, JpaSpecificationExecutor, 동적 쿼리, dynamic query, 선택 조건, optional condition, 조합 폭발, 2의 n승, is null or, JPQL null 체크, 조건 조립, and, 필드 이름 문자열, root.get, 런타임 에러, IllegalArgumentException, 정적 메타모델, static metamodel, Querydsl, BooleanBuilder, 컴파일 타임 검사, PlanSpecifications, PostRepository, study-board, 면접 답변]
topic: 선택 조건이 여러 개 자유롭게 조합되면 Specification 으로 조건을 조각내 이어 붙인다(5개 → 2⁵=32 조합). 선택 조건이 하나면 JPQL + is null or 로 충분. 약점은 필드 이름이 문자열이라 컴파일 때 못 잡는 것
summary: 계획 검색은 키워드·카테고리·완료 여부·시작일·종료일 다섯 조건이 있거나 없을 수 있어 조합이 2⁵=32 가지다. 조합마다 JPQL 을 만들거나 :param is null or 를 다섯 줄 늘어놓는 대신, PlanSpecifications 가 조건 하나를 메서드 하나로 떼어 있을 때만 and() 로 이어 붙인다. 게시글 검색은 검색 종류를 메서드 4개로 가르고 선택 조건은 카테고리 하나뿐이라 JPQL 의 is null or 한 줄로 충분했다. Criteria 는 JPQL 을 자바로 조립하는 객체 레벨 API 라 root.get("title") 의 "title" 은 DB 컬럼이 아니라 엔티티 필드 이름 문자열이고, 이름이 바뀌어도 컴파일러가 못 잡아 실행 시 에러가 난다 → 정적 메타모델(Plan_.title)이나 Querydsl 로 컴파일 시점 검사.
source: session
distilled: true
distilled_at: 2026-10-18
---

## 배운 개념

### 1. 계획 검색 — 선택 조건 5개

```java
// PlanSpecifications — 조건 하나 = 메서드 하나, 있을 때만 이어 붙임
public static Specification<Plan> matching(PlanSearchCondition condition) {
    Specification<Plan> spec = (root, query, cb) -> cb.conjunction();
    if (condition.hasKeyword())  spec = spec.and(titleOrContentContains(condition.keyword()));
    if (condition.hasCategory()) spec = spec.and((root, q, cb) -> cb.equal(root.get("category"), condition.category()));
    if (condition.completedFilter() != null) spec = spec.and(...);
    if (condition.from() != null) spec = spec.and(...);
    if (condition.to() != null)   spec = spec.and(...);
    return spec;
}

// PlanService
planRepository.findAll(PlanSpecifications.ownedBy(memberId).and(PlanSpecifications.matching(condition)), ...);
```

```
키워드 × 카테고리 × 완료 여부 × 시작일 × 종료일
  2    ×    2     ×    2     ×   2   ×   2    = 2⁵ = 32
```

- 조합마다 JPQL → 메서드 32개. `:param is null or ...` 로 몰아넣기 → null 체크 5줄 + 파라미터 타입 문제. 조건 하나 늘 때마다 두 배.
- `ownedBy(memberId)` 는 "남의 계획이 섞이지 않도록 항상 먼저" 거는 조각.

### 2. 게시글 검색 — 선택 조건 1개라 JPQL 로 충분

```java
findSummaries(category, pageable)                     // 검색어 없음
findSummariesByTitle(keyword, category, ...)          // 제목
findSummariesByTitleOrContent(keyword, category, ...) // 제목+내용
findSummariesByAuthorNickname(keyword, category, ...) // 작성자
// CATEGORY_FILTER = " and (:category is null or p.category = :category)"
```

- 검색 **종류**는 사용자가 하나만 고르니 메서드 4개로 분리, 그 위의 선택 조건은 **카테고리 하나**.

> **기준**: 선택 조건이 한두 개면 JPQL + `is null or`, 여러 개가 자유롭게 조합되면 Specification.

### 3. 약점 — 필드 이름이 문자열

```java
cb.like(cb.lower(root.get("title")), pattern)
```

- Criteria 는 JPQL 을 자바로 조립하는 **객체 레벨** API → `"title"` 은 **엔티티 필드 이름** (컬럼이 `plan_title` 이어도 `"title"`).
- 그냥 `String` 이라 컴파일러가 검사 못 함 → 필드 이름을 바꾸고 이 줄을 놓치면 **실행해서 검색할 때** 에러.

| 방법 | 코드 | 필드 이름이 바뀌면 |
|---|---|---|
| 문자열 (현재) | `root.get("title")` | **실행 시** 에러 |
| JPA 정적 메타모델 | `root.get(Plan_.title)` | **컴파일** 에러 (`Plan_` 은 빌드 때 생성) |
| Querydsl | `QPlan.plan.title.containsIgnoreCase(kw)` | **컴파일** 에러 + 가독성 |

- study-board 는 Specification 이 한 곳뿐이라 문자열로도 감당 가능. 동적 쿼리가 늘면 Querydsl 검토.

### 4. 면접 답변 (30초, 세 방식 통합)

> "기본은 JPQL입니다. 엔티티와 필드를 대상으로 쓰고, Hibernate가 Dialect에 맞춰 SQL로 번역해 주니 H2에서 PostgreSQL로 바꿔도 그대로 동작합니다. study-board의 @Query 26개가 전부 JPQL입니다.
> Native는 JPQL로 표현할 수 없거나 막힐 때만 씁니다. 전체 공개 계획을 올리면 회원 수만큼 알림을 만들어야 하는데, save()를 반복하면 IDENTITY라 회원 만 명에 INSERT 왕복이 만 번이고, JPQL insert-select는 Hibernate가 IDENTITY 엔티티에 막아서 Native insert-select 한 문장으로 처리했습니다. 대신 DB에 묶이고 영속성 컨텍스트를 거치지 않으니 clearAutomatically를 붙였습니다.
> Specification은 선택 조건이 여러 개 자유롭게 조합될 때 씁니다. 계획 검색은 조건이 다섯 개라 조합이 32가지여서, 조건 하나를 메서드 하나로 떼고 있을 때만 and로 이어 붙였습니다. 게시글 검색처럼 선택 조건이 하나뿐이면 JPQL로 충분합니다. 약점은 root.get("title")처럼 필드 이름이 문자열이라 이름이 바뀌어도 컴파일 때 잡히지 않는다는 점이고, 동적 쿼리가 늘면 정적 메타모델이나 Querydsl로 옮기겠습니다."

## 오답·헷갈린 점

- **전**: 선택 조건 5개의 조합 = 25 → **후**: 있다/없다 2가지씩 독립 → **2⁵ = 32** (5² 로 셈).
- **전**: 게시글 검색의 선택 조건 = 5개 → **후**: **1개**(카테고리). 검색 종류는 메서드 4개로 분리. "단일 검색이라 충분" 직감은 맞음 → 숫자로 "선택 조건이 하나라서".
- **전**: `root.get("title")` 의 `"title"` = "DB 컬럼 이름" → **후**: **엔티티 필드 이름** 문자열. Criteria 는 객체 레벨.
  - ⚠️ **객체 레벨 vs DB 레벨 혼동 — 이번 주 반복 테마** (10/13 프록시 "조인", 10/16 cascade "테이블 CRUD", 10/17 주인 "테이블", 10/18 Criteria "컬럼"). **JPQL·Criteria 는 객체 레벨, Native 만 DB 레벨.**
- ✅ "실행 시 에러", "다수 조건에서 Specification" 은 스스로 답함.

## Q&A

- **Q. 선택 조건 5개의 조합 수?** A. 32.
- **Q. 게시글 검색은 왜 JPQL 로 충분?** A. 선택 조건이 카테고리 하나뿐.
- **Q. `root.get("title")` 의 오타·이름 변경은 언제 드러나나?** A. 실행 시. 문자열이라 컴파일러가 못 잡음.
- **Q. 해결책?** A. 정적 메타모델, Querydsl.

## 다음 학습

- 같은 계획 검색을 Querydsl `BooleanBuilder` / `BooleanExpression` 으로 옮겨 보기 (+ Querydsl 유지보수 현황 확인)
- `hibernate-jpamodelgen` 으로 정적 메타모델 생성해 보기
- 연결: [[2026-10-18-jpql-vs-native-query]], [[derived-query-method]], [[like-search-btree-index]]
