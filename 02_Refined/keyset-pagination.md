---
type: refined
slug: keyset-pagination
tags: [키셋페이징, keyset-pagination, 커서페이징, cursor-pagination, seek-method, OFFSET성능, offset-절벽, deep-pagination, 무한스크롤, infinite-scroll, 인덱스탐색, index-seek, LIMIT, 마지막본값, no-offset]
topic: OFFSET 이 깊어질수록 느려지는 이유와 "마지막으로 본 값 다음부터" 로 바꾸는 처방
summary: OFFSET 은 점프가 아니라 앞의 N건을 읽고 버리는 동작이라 페이지 깊이에 비례해 느려진다. 키셋 페이징은 정렬 키의 마지막 값을 WHERE 조건으로 넘겨 인덱스로 바로 착지하므로 몇 페이지든 비용이 일정하다.
contributors: [dongju]
source_refs:
  - https://use-the-index-luke.com/sql/partial-results/fetch-next-page
updated: 2026-08-25
---

# 키셋(커서) 페이징

## 개념 정의

페이지를 **"몇 번째부터"(offset)** 가 아니라 **"마지막으로 본 값 다음부터"(keyset)** 로 지정하는 방식.
seek method, cursor pagination, no-offset 이라고도 부른다.

## 작동 원리

### OFFSET 의 성능 절벽

```sql
SELECT * FROM post ORDER BY id DESC LIMIT 10 OFFSET 999980;
```

`OFFSET 999980` 은 **점프가 아니다.** DB 는 인덱스를 따라 **999,980건을 실제로 읽어서 버린 뒤**
다음 10건을 돌려준다.

- 비용이 **페이지 깊이에 선형 비례**한다. 1페이지는 즉시, 마지막 페이지는 사실상 테이블 전체 읽기.
- 앞부분만 테스트하면 절대 드러나지 않는다. 크롤러나 "마지막 페이지" 클릭이 운영에서 이걸 깨운다.

### 키셋으로 바꾸기

```sql
-- offset 방식: 999,980건 읽고 버림
SELECT * FROM post ORDER BY id DESC LIMIT 10 OFFSET 999980;

-- 키셋 방식: 인덱스로 id=20 지점에 바로 착지
SELECT * FROM post WHERE id < 20 ORDER BY id DESC LIMIT 10;
```

클라이언트는 직전 페이지의 **마지막 행의 정렬 키 값(커서)** 을 들고 와서 다음 요청에 실어 보낸다.
DB 는 B-tree 인덱스에서 그 지점을 바로 찾아(seek) 10건만 읽는다 → **몇 페이지든 비용이 같다.**

정렬 키가 유일하지 않으면 **복합 커서**가 필요하다.

```sql
WHERE (created_at, id) < (:lastCreatedAt, :lastId)
ORDER BY created_at DESC, id DESC LIMIT 10
```

## 트레이드오프 / 한계

| | OFFSET | 키셋 |
|---|---|---|
| 깊은 페이지 성능 | 선형 악화 | 일정 |
| 임의 페이지 점프("7페이지로") | 가능 | **불가능** |
| 전체 페이지 수 표시 | 가능(count 비용) | 보통 포기 |
| 정렬 기준 변경 | 자유로움 | 커서 설계를 다시 해야 함 |
| 삽입/삭제 중 중복·누락 | 발생 가능 | 구조적으로 안전 |

**"임의 페이지 점프를 포기하는 대신 깊이에 무관한 성능을 얻는" 교환.**
무한 스크롤·타임라인·피드 UI 가 대부분 이 방식인 이유이며, 반대로 페이지 번호 네비게이션이 필수인
관리자 화면에서는 OFFSET 을 유지하고 대신 검색 조건으로 결과 집합을 줄이는 편이 현실적이다.

Spring Data JPA 에서는 파생 쿼리로도 표현된다: `findByIdLessThanOrderByIdDesc(Long cursor, Pageable pageable)`
([[derived-query-method]]). 반환 타입은 count 가 필요 없으므로 `Slice` 가 자연스럽다 ([[db-pagination]]).

> [!WARNING]
> **오답 코너**
> - **"OFFSET 은 해당 위치로 점프한다"** — 앞의 N건을 **읽고 버린다**. 그래서 깊어질수록 느리다.
> - **"인덱스를 잘 걸면 OFFSET 도 빠르다"** — 인덱스가 있어도 건너뛸 행을 세면서 읽어야 하므로 해결되지 않는다.
> - **"키셋은 무조건 우월하다"** — 임의 페이지 점프와 전체 페이지 수를 포기하는 교환이다.
> - **"커서는 페이지 번호를 넘기면 된다"** — 커서는 **마지막으로 본 행의 정렬 키 값**이다.

## 복습 체크

- [ ] `OFFSET 999980` 이 느린 이유를 DB 동작 수준에서 설명할 수 있는가?
- [ ] 같은 페이지를 키셋 방식 SQL 로 바꿔 쓸 수 있는가?
- [ ] 커서로 넘기는 값이 정확히 무엇인가?
- [ ] 정렬 키가 유일하지 않을 때 커서를 어떻게 구성하는가?
- [ ] 키셋이 포기하는 기능 두 가지를 댈 수 있는가?
- [ ] 어떤 UI 에 키셋이 적합하고, 어떤 화면에는 OFFSET 을 유지하는 게 나은가?

## 관련

[[db-pagination]] · [[like-search-btree-index]] · [[derived-query-method]]
