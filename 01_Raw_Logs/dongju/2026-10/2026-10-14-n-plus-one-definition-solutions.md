---
date: 2026-10-14
type: raw
author: dongju
tags: [N+1, n plus 1, N+1 문제, 정의, definition, MyBatis, 접근 패턴, access pattern, 해결책, fetch join, join fetch, 페치 조인, EntityGraph, default_batch_fetch_size, batch size, BatchSize, in 쿼리, DTO 조회, DTO projection, select new, 프로젝션, projection, Object[], 반정규화, denormalization, 외래 키, FK, 면접 답변]
topic: N+1 의 정확한 정의(1 과 N 이 무엇인가, JPA 만의 문제가 아닌 이유)와 해결책 다섯 가지의 쿼리 수·결과 타입 비교
summary: N+1 은 목록 쿼리 1번 뒤에 결과 행마다 연관 데이터를 따로 조회하는 쿼리가 N번 더 나가는 문제로, 원인은 행 단위 조회라는 접근 패턴이라 MyBatis 에서도 생긴다. 해결책은 join fetch·@EntityGraph(1번, 엔티티), batch fetch size(1+1번, 엔티티), select new DTO 조회(1번, DTO)이고, 반정규화는 연관 조회 자체를 없애는 설계 선택으로 따로 분류한다.
source: session
distilled: true
distilled_at: 2026-10-14
---

## 배운 개념

### 1. 정의

> **N+1 문제**는 목록을 조회하는 쿼리 **1번**을 실행한 뒤, 결과 행마다 연관 데이터를 FK 로 하나씩 조회하는 쿼리가 **N번** 더 나가는 문제다. 연관 데이터를 첫 쿼리에서 함께 가져오지 않고 **행 단위로 따로** 가져오기 때문에 생긴다.

- **1** = 목록(게시글)을 가져오는 쿼리.
- **N** = 결과 행 수만큼 나가는 연관 데이터(작성자) 조회 쿼리. JPA 에서는 1차 캐시 덕분에 **서로 다른 연관 엔티티 수**로 줄어든다.
- 정의에 EAGER/LAZY 를 넣으면 안 된다 — 두 설정 모두에서 생기므로 fetch 설정은 원인이 아니다.

### 2. JPA 만의 문제가 아니다

```java
List<Post> posts = postMapper.findAll();                         // select * from post
for (Post post : posts) {
    Member m = memberMapper.findById(post.getMemberId());       // select * from member where id = ?
}
```

- 프록시도 EAGER 도 없는데 똑같이 1 + N.
- N+1 은 **데이터 접근 패턴**의 문제. JPA 는 이 루프가 `getName()` 한 줄 뒤에 **숨어서 안 보일** 뿐이다.

### 3. 해결책 비교

| 방법 | 쿼리 수 | 결과 | 분류 |
|---|---|---|---|
| `join fetch` | 1 | 엔티티 | 쿼리 기법 (JPQL 에 직접) |
| `@EntityGraph(attributePaths = "member")` | 1 | 엔티티 | 쿼리 기법 (어노테이션으로 fetch join) |
| `default_batch_fetch_size` | 1+1 | 엔티티 | 프록시 초기화 방식 변경 |
| `select new` DTO 조회 | 1 | DTO | 쿼리 기법 |
| 반정규화 (Post 에 닉네임 복사) | 1 | 엔티티 | **설계 선택** — 연관 조회 자체를 없앰 |

### 4. batch fetch size 는 조회 쿼리를 바꾸지 않는다

```
① findAll()          → select * from post                       (1번, member 는 프록시)
② 첫 getName() 호출   → 설정 없으면: where id = ?                (프록시 하나만 초기화)
                        설정 있으면: where id in (?, ?, ..., ?)   (아직 초기화 안 된 프록시를 모아 한 번에)
```

- 총 1 + 1 = **2번**. 초기화되는 건 프록시가 가리키던 진짜 `Member` → 결과는 **엔티티**.
- 연관 엔티티 수가 배치 크기보다 많으면 `in` 쿼리가 몇 번 더 나갈 수 있다.

### 5. DTO 조회 — `select new`

```java
@Query("select new com.example.board.post.dto.PostListItem(p.id, p.title, m.name, p.createdAt) " +
       "from Post p join p.member m")
List<PostListItem> findListItems();
```

- 값만 나열(`select p.id, p.title, ...`)하면 행마다 `Object[]` 로 와서 직접 꺼내고 캐스팅해야 한다.
- `new` + **패키지 포함 전체 클래스명** + 일치하는 생성자 → DTO 로 바로 받음.
- DTO 는 영속성 컨텍스트가 관리하지 않는다 → **프록시·지연 로딩·변경 감지 모두 없음**.

## 오답·헷갈린 점

- **전**: "쿼리가 결과의 개수만큼 1번 더 조회" (2번인지 1+N번인지 모호) → **후**: 결과 **행마다** 1번씩 → 1 + N.
- **전**: "엔티티에 조인된 테이블이 있어서 같이 가져와야 하기 때문에" 발생 → **후**: 반대. 조인해서 같이 가져오면 **해결**(1번). 같이 가져오지 **않고** 행마다 따로 가져와서 발생.
- **전**: "ID 외의 데이터를 읽을 때 한 번 더 조회가 필요해서" → **후**: LAZY 프록시에서만 맞는 설명. EAGER·MyBatis 의 N+1 을 설명하지 못함 → 공통점은 "행 단위로 따로 조회".
- **전**: FK 를 "시퀀스값"이라 부름 → **후**: 다른 테이블 행을 가리키는 값은 **외래 키(FK)**. 시퀀스는 PK 생성기(`nextval`).
- **전**: `left fetch join` → **후**: **`left join fetch`** (join 다음에 fetch).
- **전**: 반정규화를 "member 값을 id 로 묶는다"로 표현, 쿼리 기법과 같은 줄에 나열 → **후**: 연관 조회를 없애는 **설계 선택**으로 따로 분류. 대가(정합성 책임)가 따른다 → [[2026-10-14-denormalized-counter-lost-update]].
- **전**: batch fetch size = "1번, DTO" → **후**: **1+1번, 엔티티**. 조회 쿼리가 아니라 프록시 초기화 방식을 바꿈.
- **전**: DTO 조회 = "`m.name` 으로 바로 값을 받으면 된다" → **후**: 값 나열은 `Object[]`. **`select new 패키지.클래스(...)`** 필요.
- ⚠️ 반복 테마: 정의를 특정 구현(LAZY 프록시)의 동작으로 설명 → 여러 경우를 포괄하는 공통 원인으로 일반화하는 연습 필요.

## Q&A

- **Q. N+1 을 한 문장으로 (EAGER/LAZY 금지)?** A. 목록 쿼리 1번 뒤 결과 행마다 연관 데이터를 따로 조회하는 쿼리가 N번 더 나가는 문제.
- **Q. MyBatis 에서도 생기나?** A. 생긴다. 접근 패턴의 문제. JPA 는 루프가 숨어 있을 뿐.
- **Q. 해결책과 쿼리 수?** A. join fetch 1, @EntityGraph 1, batch size 1+1, DTO 1. (반정규화는 설계 선택)
- **Q. batch size 의 결과 타입?** A. 엔티티 (초기화된 진짜 Member).
- **Q. 엔티티 대신 화면용 값만 받으려면?** A. `select new` + 전체 클래스명.

## 다음 학습

- `show-sql` 로 batch size 적용 전후 `in` 쿼리 직접 확인
- `@Fetch(FetchMode.SUBSELECT)` 와 batch size 비교
- 연결: [[n-plus-1-problem]], [[lazy-loading-proxy]], [[2026-10-13-eager-jpql-n-plus-one]], [[2026-10-14-collection-join-paging-dto-entitygraph]]
