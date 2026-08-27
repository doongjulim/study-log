---
type: refined
slug: derived-query-method
tags: [파생쿼리, derived-query, 쿼리메서드, query-method, 메서드이름쿼리, findBy, Containing, StartingWith, IgnoreCase, PartTree, 구동시점검증, fail-fast, 빈생성예외, 엔티티필드명, Spring-Data-JPA]
topic: 메서드 이름이 쿼리로 번역되는 규칙과, 그 파싱이 "구동 시점"에 일어난다는 것의 가치
summary: findBy + 엔티티 필드명 + 조건 키워드로 이루어진 메서드 이름을 Spring Data 가 구동 시점에 파싱해 쿼리로 만든다. 오타는 컴파일러가 못 잡지만 구동 시 빈 생성 예외로 즉시 드러나 실행 시점 오류를 앞당겨 준다.
contributors: [dongju]
source_refs:
  - https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html
updated: 2026-08-25
---

# 파생 쿼리 메서드 (Derived Query)

## 개념 정의

메서드 **이름 자체가 쿼리 선언**이 되는 Spring Data 의 기능.

```java
Page<Post> findByTitleContainingIgnoreCase(String keyword, Pageable pageable);
```

| 조각 | 의미 |
|---|---|
| `findBy` | 조회 시작(주어부). `countBy`, `existsBy`, `deleteBy` 도 있음 |
| `Title` | **엔티티의 필드명** — DB 컬럼명이 아니다. 매핑은 JPA 메타데이터가 해석 |
| `Containing` | `LIKE '%keyword%'` |
| `IgnoreCase` | 양쪽에 `lower()` 적용 |
| `Pageable` 파라미터 | `LIMIT`/`OFFSET`/`ORDER BY` 로 번역 |

자주 쓰는 키워드: `And`, `Or`, `Between`, `LessThan`, `GreaterThanEqual`, `Like`, `StartingWith`,
`EndingWith`, `In`, `IsNull`, `OrderBy...Desc`.

## 작동 원리

파싱은 **앱 구동 시점**에 일어난다. 리포지토리 프록시를 만들 때 `PartTree` 가 메서드 이름을
토큰으로 쪼개 쿼리를 미리 만들어 둔다 ([[spring-data-repository-proxy]]).

**이 시점이 이 기능의 숨은 가치다.**

```java
Page<Post> findByTitel(String kw);   // 오타
```

- **컴파일은 통과한다** — 자바 문법상 완벽히 합법적인 메서드 이름이다. 컴파일러는 이게 쿼리인 줄 모른다.
- **구동 시 서버가 아예 안 뜬다** — `Title` 이라는 필드를 못 찾아 빈 생성 예외
  (`PropertyReferenceException: No property 'titel' found for type 'Post'`).

즉 **런타임 오류를 구동 시점으로 앞당기는(fail-fast)** 장치다. 배포 직후 서버가 안 뜨는 것이
"운영 중 특정 화면에서만 500 이 나는 것"보다 훨씬 낫다.

## 트레이드오프 / 한계

- **이름이 길어지면 가독성이 무너진다.** 조건이 3개를 넘어가면
  `findByTitleContainingAndCategoryAndCreatedAtBetweenOrderByCreatedAtDesc` 같은 괴물이 된다.
  → 이 지점부터는 `@Query`(JPQL), Querydsl, Specification 으로 넘어가는 것이 맞다.
- **동적 조건에 약하다.** "검색어가 있을 때만 조건 추가" 같은 분기는 메서드 이름으로 표현할 수 없다.
  → Querydsl `BooleanBuilder` 나 Specification.
- **필드명 리팩터링에 취약하다.** 엔티티 필드명을 바꾸면 메서드 이름도 같이 바꿔야 하며,
  IDE 자동 리팩터링이 문자열 규약을 항상 따라가 주지는 않는다. (구동 시 즉시 드러나는 것이 방어막)
- 이름이 만들어 내는 SQL 의 **성능 특성까지 감춰진다** — `Containing` 은 앞 와일드카드라
  인덱스를 못 탄다는 사실이 이름에는 드러나지 않는다 ([[like-search-btree-index]]).

> [!WARNING]
> **오답 코너**
> - **"파생 쿼리 오타는 컴파일 시 잡힌다"** — 자바 문법상 합법이라 컴파일러는 모른다.
>   **앱 구동 시** 메서드명 파싱이 실패하며 빈 생성 예외가 난다.
> - **"메서드 이름에 쓰는 건 DB 컬럼명"** — **엔티티 필드명**이다. 컬럼 매핑은 JPA 가 해석한다.
> - **"파싱은 메서드가 호출될 때마다 일어난다"** — 구동 시 1회다.
> - **"이름으로 다 되니까 `@Query` 는 불필요"** — 조건이 늘거나 동적이면 즉시 한계에 부딪힌다.

## 복습 체크

- [ ] `findByTitleContainingIgnoreCase` 를 조각으로 쪼개 각각의 의미를 말할 수 있는가?
- [ ] 메서드 이름에 쓰는 것이 필드명인지 컬럼명인지 구분할 수 있는가?
- [ ] 파싱이 언제 일어나는가? 그것이 왜 장점인가?
- [ ] `findByTitel` 오타는 언제, 어떤 형태로 드러나는가?
- [ ] 파생 쿼리를 버리고 `@Query`/Querydsl 로 넘어가야 할 신호 두 가지는?

## 관련

[[spring-data-repository-proxy]] · [[like-search-btree-index]] · [[db-pagination]] · [[keyset-pagination]]
