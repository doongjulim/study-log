---
date: 2026-07-11
type: raw
author: dongju
tags: [Pageable, Page, Slice, 페이징, paging, pagination, 검색, search, offset, limit, 키셋페이징, keyset-pagination, cursor-pagination, 무한스크롤, count-query, LIKE검색, B-tree, 인덱스, index, full-scan, 파생쿼리, derived-query, 동적프록시, dynamic-proxy, ArgumentResolver, PageableDefault, 스냅샷, snapshot, readOnly]
topic: Spring Data JPA 페이징 & 검색(Pageable)의 작동 원리와 한계 (study-board 프로젝트)
summary: 개인 프로젝트 study-board의 게시글 목록/검색을 소재로, DB 페이징이 필요한 이유(메모리·스냅샷), Pageable이 만들어지고 실행되는 메커니즘(ArgumentResolver·구동 시 파싱·동적 프록시), offset 성능 절벽과 LIKE '%kw%' 인덱스 불가·Page vs Slice 트레이드오프를 인터뷰로 검증했다.
source: session
distilled: false
---

# Spring Data JPA 페이징 & 검색(Pageable)의 작동 원리와 한계

개인 프로젝트 study-board(Spring Boot 3.3 게시판)의 `PostController.list()` / `PostRepository` 검색 메서드를 소재로 한 study-interview 세션.

## 배운 개념

### 왜 DB 수준 페이징인가 (What & Why)

- `findAll()`로 전부 가져와 애플리케이션에서 자르면 부담이 세 군데에 걸린다:
  1. **DB 디스크 읽기** — 전체를 읽는다.
  2. **네트워크 전송** — DB→앱 서버로 수백 MB가 소켓을 타고 흘러야 하고, 그동안 커넥션이 점유된다.
  3. **JVM 힙** — 100만 개의 엔티티 객체가 쌓여 OOM으로 이어질 수 있다.
- JPA면 3번이 더 심해진다: 영속성 컨텍스트가 변경 감지(dirty checking)용 **스냅샷**(조회 시점 상태의 복사본)을 엔티티마다 하나씩 더 만들어서 메모리 부담이 대략 두 배.
- `@Transactional(readOnly = true)`는 관례가 아니라 최적화 — Hibernate가 **스냅샷 생성과 flush를 생략**한다.
- 페이징 SQL의 최소 형태: `SELECT ... ORDER BY ... LIMIT 10 OFFSET 20` (3페이지, 페이지당 10건 기준).
- `ORDER BY`는 장식이 아니라 **전제조건**. 페이징 = "전체를 같은 기준으로 줄 세운 뒤 자르기"라서, 정렬이 없으면 DB가 순서를 보장하지 않아 페이지를 넘기는 사이 같은 글이 **중복** 노출되거나 **누락**될 수 있다.
- 반환 타입이 `Page<T>`인 순간 Spring Data JPA는 `getTotalPages()`용으로 `SELECT COUNT(*)` 쿼리를 **한 번 더** 날린다 (본 쿼리 + 카운트 쿼리, 총 2방).

### Pageable이 만들어지는 곳 — ArgumentResolver

```java
public String list(..., @PageableDefault(size = 10) Pageable pageable, ...)
```

- 컨트롤러에서 `Pageable`을 직접 만든 적이 없는데 채워져 있는 이유: Spring MVC는 컨트롤러 메서드 호출 직전에 파라미터마다 담당 **`HandlerMethodArgumentResolver`** 를 찾아 값을 채운다 (`@RequestParam`, `Model`과 같은 계층의 메커니즘).
- `Pageable` 전용으로 **`PageableHandlerMethodArgumentResolver`** 가 등록되어 있고(Boot 자동 구성), 쿼리스트링의 `page`/`size`/`sort`를 읽어 `PageRequest`(Pageable 구현체)를 만든다.
- 파라미터가 없으면 `@PageableDefault` 값으로 채운다. `/posts?size=30`처럼 page 없이 오면 `PageRequest.of(0, 30)` — 첫 페이지.

### 파생 쿼리 메서드 — 이름이 쿼리가 되는 시점

```java
Page<Post> findByTitleContainingIgnoreCase(String keyword, Pageable pageable);
```

- `findBy`(조회 시작) / `Title`(**엔티티 필드명** — DB 컬럼명이 아님, 매핑은 JPA 메타데이터가 해석) / `Containing`(`LIKE '%kw%'`) / `IgnoreCase`(양쪽 `lower()`).
- 파싱은 **앱 구동 시점**에 일어난다. `findByTitel`처럼 오타를 내면 컴파일은 통과하지만(자바 문법상 합법적 이름) 구동 시 빈 생성 예외로 서버가 아예 안 뜬다 → **실행 시점 오류를 구동 시점으로 앞당기는** 것이 파생 쿼리의 숨은 가치.

### 구현 없는 인터페이스가 실행되는 이유 — 동적 프록시

- 앱 구동 시 딱 한 번, Spring Data JPA가 리포지토리 인터페이스를 발견하고 그것을 구현하는 클래스를 **런타임에 즉석 생성**(JDK 동적 프록시)해서 **싱글턴 빈**으로 등록·주입한다. 요청마다 만드는 게 아니다.
- 기본 메서드(`findAll` 등)는 `SimpleJpaRepository`로 위임, 파생 메서드는 구동 시 파싱해 둔 쿼리를 실행.
- `@Transactional`이 트랜잭션을 자동으로 여닫는 것도 같은 원리: **프록시가 호출을 가로채 부가 기능(트랜잭션 시작 → 위임 → 커밋/롤백)을 수행**한다.

### 트레이드오프와 한계 (Edge Cases)

**① offset의 성능 절벽**

- `OFFSET 999980`은 "점프"가 아니다. DB는 인덱스를 따라 **999,980건을 읽어서 버린 뒤** 다음 10건을 준다 → 뒤 페이지일수록 비용이 선형 증가, 마지막 페이지는 사실상 테이블 전체 읽기.
- 처방: **키셋(keyset/cursor) 페이징** — "몇 번째부터" 대신 "마지막으로 본 값 다음부터".

```sql
-- offset 방식: 999,980건 읽고 버림
... ORDER BY id DESC LIMIT 10 OFFSET 999980
-- 키셋 방식: 인덱스로 id=20인 지점에 바로 착지
WHERE id < 20 ORDER BY id DESC LIMIT 10
```

- 키셋은 몇 페이지든 비용이 같지만 "임의 페이지 점프"를 포기하는 트레이드오프. 무한 스크롤 UI가 대부분 이 방식.

**② `LIKE '%kw%'`는 인덱스를 못 탄다**

- 일반 DB 인덱스는 **B-tree(정렬된 자료구조)** — 사전과 같다.
- `LIKE 'spring%'`(뒤 %): "spr로 시작하는 위치"로 바로 찾아 들어갈 수 있어 인덱스 사용 가능.
- `LIKE '%spring%'`(앞 %): 시작 글자를 모르니 정렬이 무용지물 → 사전 전체를 넘겨보는 **풀 스캔**. 앞 `%`가 "정렬의 시작점"을 지워버리는 것이 원인.
- 처방: DB 전문 검색 인덱스(full-text) 또는 Elasticsearch 같은 검색 엔진 분리.

**③ Page vs Slice — count 쿼리의 청구서**

- `Page`는 목록 조회마다 `COUNT(*)`를 추가로 지불한다. 대용량 테이블에선 count도 공짜가 아니다.
- 무한 스크롤이라면 "전체 몇 페이지"가 필요 없으므로 **`Slice<T>`** — count 없이 **limit+1 조회**(10건 필요하면 11건 조회)로 11번째 존재 여부만 보고 "다음 페이지 있음"을 판단한다.
- 반환 타입 선택이 곧 쿼리 비용 선택: 페이지 번호 UI → `Page`, 더보기/무한 스크롤 → `Slice`.

## 오답·헷갈린 점

- **"데모 객체" → 스냅샷**: 영속성 컨텍스트가 변경 감지용으로 두는 것의 이름을 잘못 기억. 조회 시점 상태의 복사본(스냅샷)이며 flush 때 현재 엔티티와 비교해 UPDATE를 만든다.
- **"파생 쿼리 오타는 컴파일 시 발견" → 앱 구동 시 발견**: `findByTitel`은 자바 문법상 합법이라 컴파일러는 모른다. 구동 시 메서드명 파싱이 실패하며 빈 생성 예외.
- **"리포지토리 구현체 = 요청마다 DB 커넥션 생성" → 구동 시 1회 생성되는 동적 프록시 싱글턴 빈**: DB 커넥션은 쿼리 실행 시 커넥션 풀에서 빌리는 별개 계층의 자원. 질문의 답은 프록시.
- **"필터가 가로챈다" → 프록시가 가로챈다**: 필터는 서블릿 계층에서 HTTP 요청을 가로채는 별개 개념.
- **"`%kw%`가 인덱스를 못 타는 건 hashcode 변환 때문" → B-tree 정렬 구조 때문**: 인덱스는 해시가 아니라 정렬 자료구조이고, 앞 `%`가 탐색 시작점을 지워서 못 타는 것.

## Q&A

- Q. 100만 건을 `findAll()`로 가져오면 어디가 부담인가?
  A. DB 읽기 + 네트워크 전송 + JVM 힙(OOM). JPA는 스냅샷 때문에 메모리 부담이 두 배.
- Q. 3페이지(페이지당 10건)의 SQL은?
  A. `... ORDER BY ... LIMIT 10 OFFSET 20`. `Page` 반환이면 `COUNT(*)` 쿼리가 하나 더 나간다.
- Q. `ORDER BY` 없이 페이징하면?
  A. DB가 순서를 보장하지 않아 페이지 간 중복/누락 발생. 페이징은 "같은 기준으로 줄 세운 뒤 자르기"라 정렬이 전제조건.
- Q. `?page=2&size=10`이 어떻게 `Pageable`이 되나?
  A. `PageableHandlerMethodArgumentResolver`가 쿼리스트링을 읽어 `PageRequest`를 생성, 없으면 `@PageableDefault` 값 사용.
- Q. 인터페이스뿐인 리포지토리가 어떻게 실행되나?
  A. 구동 시 Spring Data가 동적 프록시를 만들어 싱글턴 빈으로 등록. 메서드명 파싱도 이때 일어난다.
- Q. `OFFSET 999980`은 왜 느린가?
  A. 점프가 아니라 999,980건을 읽어 버리는 방식이라 페이지 깊이에 비례해 느려진다. 키셋 페이징(`WHERE id < 마지막값`)이 처방.
- Q. count 쿼리를 생략하려면?
  A. `Slice` — limit+1 조회로 다음 페이지 유무만 판단. 무한 스크롤 UI에 적합.

## 다음 학습

- 키셋 페이징을 study-board에 실제 적용해 보기 (무한 스크롤 목록 + `WHERE id < ?` 리포지토리 메서드)
- `Page`의 count 쿼리 최적화: `@Query(countQuery = ...)` 분리, join 시 count 쿼리가 무거워지는 문제
- H2/MySQL에서 `EXPLAIN`으로 `LIKE 'kw%'` vs `LIKE '%kw%'` 실행 계획 직접 비교
- 전문 검색(full-text index) 또는 Elasticsearch 연동 개요
