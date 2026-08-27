---
type: refined
slug: db-pagination
tags: [페이징, paging, pagination, DB페이징, LIMIT, OFFSET, ORDER-BY, 정렬전제, 중복노출, 누락, Page, Slice, count-query, COUNT쿼리, getTotalPages, 무한스크롤, limit-plus-1, PageRequest, Pageable, readOnly, 스냅샷, JPA, Spring-Data-JPA]
topic: 애플리케이션이 아니라 DB 에서 잘라야 하는 이유와 Page/Slice 의 쿼리 비용 차이
summary: 전부 읽어 애플리케이션에서 자르면 DB 읽기·네트워크·힙 세 군데가 동시에 무너진다. DB 페이징은 ORDER BY 를 전제로 하며, 반환 타입(Page vs Slice) 선택이 곧 count 쿼리 비용의 선택이다.
contributors: [dongju]
source_refs:
  - https://docs.spring.io/spring-data/jpa/reference/repositories/query-methods-details.html
updated: 2026-08-25
---

# DB 수준 페이징과 Page vs Slice

## 개념 정의

**"전체를 같은 기준으로 줄 세운 뒤 필요한 구간만 자른다"** 를 DB 가 하도록 맡기는 것.

```sql
SELECT ... FROM post ORDER BY created_at DESC LIMIT 10 OFFSET 20   -- 3페이지, 10건씩
```

## 작동 원리

### 왜 애플리케이션에서 자르면 안 되는가

`findAll()` 로 100만 건을 가져와 `subList()` 하면 부담이 **세 군데**에 동시에 걸린다.

1. **DB 디스크 읽기** — 전체를 읽는다.
2. **네트워크 전송** — DB→앱 서버로 수백 MB 가 흐르고, 그동안 커넥션이 점유된다.
3. **JVM 힙** — 100만 개 엔티티 객체가 쌓여 `OutOfMemoryError` 로 이어진다
   ([[garbage-collection-reachability]] — 리스트가 참조를 붙들고 있으면 GC 도 못 치운다).

**JPA 면 3번이 더 심하다.** 영속성 컨텍스트가 변경 감지용 **스냅샷**을 엔티티마다 하나씩 더 만들어
메모리 부담이 대략 두 배가 된다 ([[dirty-checking]]).
→ 조회 전용 메서드에 `@Transactional(readOnly = true)` 를 붙이면 Hibernate 가 스냅샷 생성과 flush 를 생략한다.

### `ORDER BY` 는 장식이 아니라 전제조건

정렬이 없으면 DB 는 행 순서를 보장하지 않는다. 그러면 페이지를 넘기는 사이 같은 행이
**중복 노출**되거나 **누락**될 수 있다. 페이징의 정의 자체가 "줄 세운 뒤 자르기"이므로 정렬이 먼저다.

> 정렬 키에 **동점(tie)** 이 많으면(예: `ORDER BY created_at` 만) 같은 문제가 재발한다.
> `ORDER BY created_at DESC, id DESC` 처럼 **유일한 tie-breaker** 를 마지막에 붙이는 것이 안전하다.

### 반환 타입이 곧 쿼리 비용

| 반환 타입 | 나가는 쿼리 | 알 수 있는 것 | 적합한 UI |
|---|---|---|---|
| `Page<T>` | 본 쿼리 + **`COUNT(*)`** = 2방 | 전체 건수, 전체 페이지 수 | 페이지 번호 네비게이션 |
| `Slice<T>` | 본 쿼리 1방 (**limit + 1** 조회) | 다음 페이지 존재 여부만 | 더보기 / 무한 스크롤 |
| `List<T>` | 본 쿼리 1방 | 없음 | 개수 고정 위젯 |

`Slice` 의 원리: 10건이 필요하면 **11건을 조회**해서 11번째가 존재하는지로 "다음 페이지 있음"을 판단한다.
대용량 테이블에서 `COUNT(*)` 도 공짜가 아니므로, 전체 페이지 수가 화면에 필요 없다면 `Slice` 가 정답이다.

## 트레이드오프 / 한계

- `Page` 의 count 쿼리는 join 이 붙으면 급격히 무거워진다 → `@Query(countQuery = "...")` 로 카운트 쿼리를
  본 쿼리와 분리해 최적화할 수 있다.
- OFFSET 은 페이지가 깊어질수록 선형으로 느려진다 → [[keyset-pagination]]
- 컬렉션을 fetch join 한 채로 페이징하면 DB 페이징이 무력화되고 메모리 페이징으로 전환된다
  → [[n-plus-1-problem]]

> [!WARNING]
> **오답 코너**
> - **"어차피 앱에서 10개만 쓰니 findAll() 해도 된다"** — DB 읽기·네트워크·힙 세 군데가 모두 부담을 진다.
>   JPA 는 스냅샷 때문에 메모리가 두 배.
> - **"ORDER BY 는 보기 좋으라고 넣는다"** — 없으면 순서가 보장되지 않아 **중복·누락**이 발생한다. 전제조건이다.
> - **"`Page` 를 쓰면 쿼리가 한 방"** — `COUNT(*)` 가 한 번 더 나가 총 2방이다.
> - **"`Slice` 는 count 를 캐싱한다"** — 캐싱이 아니라 **limit+1 조회**로 다음 페이지 유무만 본다.

## 복습 체크

- [ ] 전부 가져와 앱에서 자를 때 부담이 걸리는 세 군데를 댈 수 있는가?
- [ ] JPA 에서 메모리 부담이 두 배가 되는 이유는?
- [ ] `@Transactional(readOnly = true)` 가 무엇을 생략하는가?
- [ ] `ORDER BY` 가 없으면 구체적으로 어떤 현상이 생기는가? tie-breaker 는 왜 필요한가?
- [ ] `Page` 와 `Slice` 각각 쿼리가 몇 번 나가고, 왜 그런가?
- [ ] 3페이지·10건씩의 SQL 을 직접 쓸 수 있는가?

## 관련

[[keyset-pagination]] · [[like-search-btree-index]] · [[dirty-checking]] · [[n-plus-1-problem]] · [[handler-method-argument-resolver]]
