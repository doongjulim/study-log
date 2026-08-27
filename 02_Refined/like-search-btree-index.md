---
type: refined
slug: like-search-btree-index
tags: [LIKE검색, LIKE, 와일드카드, 앞와일드카드, leading-wildcard, B-tree, 비트리인덱스, 인덱스, index, index-seek, full-scan, 풀스캔, 전문검색, full-text-search, Elasticsearch, Containing, StartingWith, 검색성능]
topic: LIKE '%kw%' 가 인덱스를 타지 못하는 이유(B-tree 는 정렬 자료구조)와 대안
summary: 인덱스는 해시가 아니라 정렬된 B-tree 라서 "시작 글자"를 알아야 탐색 지점을 찾을 수 있다. 앞 와일드카드는 그 시작점을 지워버리므로 풀 스캔이 되고, 처방은 전문 검색 인덱스나 검색 엔진 분리다.
contributors: [dongju]
source_refs:
  - https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/like-performance-tuning
updated: 2026-08-25
---

# `LIKE '%kw%'` 와 B-tree 인덱스

## 개념 정의

일반적인 DB 인덱스는 **B-tree — 정렬된 자료구조**다. 해시 테이블이 아니다.
비유하면 **사전(dictionary)** 과 같다. 정렬되어 있으므로 "시작 지점"만 알면 그리로 바로 찾아 들어갈 수 있다.

## 작동 원리

| 패턴 | 인덱스 | 이유 |
|---|---|---|
| `LIKE 'spring%'` (뒤 와일드카드) | **사용 가능** | "spr 로 시작하는 위치"로 seek 해서 그 구간만 스캔 |
| `LIKE '%spring%'` (앞 와일드카드) | **사용 불가** | 시작 글자를 모르니 정렬이 무용지물 → 사전 전체를 넘겨보는 **풀 스캔** |
| `LIKE '%spring'` | 사용 불가 | 위와 동일 |

**원인은 한 문장이다: 앞의 `%` 가 "정렬의 시작점"을 지워버린다.**
사전에서 "sp 로 시작하는 단어"는 즉시 찾지만, "가운데에 sp 가 든 단어"는 전부 넘겨봐야 하는 것과 같다.

Spring Data JPA 의 파생 쿼리에서도 그대로 드러난다 ([[derived-query-method]]).

```java
findByTitleContaining(String kw)    // LIKE '%kw%'  → 풀 스캔
findByTitleStartingWith(String kw)  // LIKE 'kw%'   → 인덱스 사용 가능
```

`IgnoreCase` 를 붙이면 양쪽에 `lower()` 가 적용되는데, **컬럼에 함수를 씌우면 그 자체로 인덱스를 못 탄다.**
대응하려면 함수 기반 인덱스(`CREATE INDEX ... ON post (lower(title))`)나 대소문자 무시 콜레이션이 필요하다.

## 트레이드오프 / 한계

**처방**

| 규모 | 방법 |
|---|---|
| 소~중 | 검색 요구를 `StartingWith` 로 축소, 다른 조건(카테고리·기간)으로 후보 집합을 먼저 줄이기 |
| 중 | DB 내장 **전문 검색 인덱스(full-text index)** — MySQL `FULLTEXT`, PostgreSQL `tsvector`/GIN |
| 대 | **검색 엔진 분리** — Elasticsearch 등. 역색인(inverted index)으로 부분 일치·형태소 분석·랭킹까지 |

전문 검색은 문자열을 **토큰(단어) 단위로 쪼개 역색인**을 만든다. 그래서 "가운데 포함"도 빠르지만,
반대로 **단어 경계를 넘는 부분 문자열**(예: `pring`)은 못 찾는다. `LIKE '%kw%'` 와 의미가 정확히 같지 않다는 점이 함정이다.

실제 실행 계획은 `EXPLAIN` 으로 확인하는 습관을 들일 것. 추측보다 확실하다.

> [!WARNING]
> **오답 코너**
> - **"`%kw%` 가 인덱스를 못 타는 건 해시 변환 때문"** — 인덱스는 해시가 아니라 **정렬(B-tree) 자료구조**다.
>   원인은 앞 `%` 가 **탐색 시작점을 지우는 것**.
> - **"인덱스만 걸면 LIKE 검색이 빨라진다"** — 앞 와일드카드면 인덱스가 있어도 무용지물이다.
> - **"`IgnoreCase` 는 성능과 무관"** — 컬럼에 `lower()` 가 씌워지면 일반 인덱스를 못 탄다.
> - **"full-text 는 `%kw%` 의 상위 호환"** — 토큰 단위 매칭이라 단어 중간 부분 문자열은 못 찾는다.

## 복습 체크

- [ ] DB 인덱스가 어떤 자료구조인지, 그 성질이 왜 중요한지 말할 수 있는가?
- [ ] `LIKE 'kw%'` 와 `LIKE '%kw%'` 의 차이를 "탐색 시작점" 개념으로 설명할 수 있는가?
- [ ] `Containing` 과 `StartingWith` 가 각각 어떤 SQL 이 되는가?
- [ ] `IgnoreCase` 가 인덱스에 미치는 영향은?
- [ ] 전문 검색 인덱스가 `LIKE '%kw%'` 와 의미상 다른 지점은?

## 관련

[[db-pagination]] · [[keyset-pagination]] · [[derived-query-method]]
