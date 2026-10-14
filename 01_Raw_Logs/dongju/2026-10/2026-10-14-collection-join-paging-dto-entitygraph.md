---
date: 2026-10-14
type: raw
author: dongju
tags: [컬렉션 조인, collection join, 컬렉션 fetch join, OneToMany, 1:N, N:1, 행 뻥튀기, row explosion, 카테시안, 페이징, paging, Pageable, limit, 메모리 페이징, in-memory paging, HHH90003004, size(), 서브쿼리, subquery, DTO 조회, select new, EntityGraph, 단건 조회, OSIV, open-in-view, LazyInitializationException, 영속성 컨텍스트 수명, study-board, 프로젝트 설계 선택]
topic: 1:N 컬렉션을 조인하면 행이 불어나 페이징이 깨진다 — study-board 가 목록은 DTO+size(), 상세는 @EntityGraph 를 고른 이유와 OSIV off 의 관계
summary: 1:N 컬렉션을 조인하면 부모 행이 자식 수만큼 불어나 DB limit 이 깨지므로 Hibernate 는 메모리 페이징으로 바꾸고 경고를 남긴다. study-board 목록은 페이징이 있어 files 를 조인하지 않고 size(p.files) 서브쿼리로 개수만 받는 select new DTO 를, 상세는 페이징 없는 단건이라 @EntityGraph 로 author·files 를 함께 로딩한다. 둘 다 OSIV off 에서 트랜잭션 안에 화면 데이터를 다 채우기 위한 선택이다.
source: session
distilled: true
distilled_at: 2026-10-14
---

## 배운 개념

### 1. 컬렉션(1:N) 조인 → 행 뻥튀기

글 2개, 파일이 각각 3개·2개일 때 `files` 를 조인하면:

```
post.id | title | file.id
--------+-------+--------
   1    | 글A   |  10
   1    | 글A   |  11      ← 글A 가 파일 수만큼 행으로 불어남
   1    | 글A   |  12
   2    | 글B   |  20
   2    | 글B   |  21
```

- 여기서 `limit 2` → 글A 행 2개만 잘림. 글B 는 빠지고 글A 파일도 2개만 → **페이지가 깨짐**.
- 그래서 Hibernate 는 **컬렉션 fetch join + 페이징**이면 DB 에서 자르지 않고 전부 메모리로 가져와 자바에서 자른다 + 경고 `HHH90003004`. 데이터가 많으면 OOM 위험.
- **N:1 조인은 안전** — 작성자는 글마다 하나라 행이 불어나지 않는다. 불어나는 건 **1:N 컬렉션**뿐.

| | 컬렉션 조인 | 페이징 | 결과 |
|---|---|---|---|
| 목록 | 행 불어남 | **있음** (`limit 20`) | 💥 페이지 깨짐 / 메모리 페이징 |
| 상세 | 행 불어남 | **없음** (`where id = ?`) | ✅ 글 하나로 합쳐지면 끝 |

### 2. study-board 실제 코드

```java
// PostRepository — 목록: select new DTO, files 는 조인하지 않고 size() 로 개수만
String SUMMARY_SELECT = "select new com.example.board.post.dto.PostSummary(" +
        "p.id, p.title, p.author.nickname, p.category, p.createdAt, size(p.files), " +
        "p.viewCount, p.likeCount) from Post p";

@Query(SUMMARY_SELECT + " where 1 = 1" + CATEGORY_FILTER)
Page<PostSummary> findSummaries(@Param("category") PostCategory category, Pageable pageable);

// 상세: 단건 → 컬렉션까지 함께
@Override
@EntityGraph(attributePaths = {"author", "files"})
Optional<Post> findById(Long id);
```

- `size(p.files)` → SQL 에서 `(select count(*) from attached_file where post_id = p.id)` 같은 **서브쿼리**. 글 하나당 숫자 하나 → 행이 불어나지 않음. 화면에 필요한 건 파일 목록이 아니라 **개수**.
- `p.author.nickname` 은 N:1 암묵적 조인 → 페이징 안전. (`author_id` 가 `not null` 이라 inner join 으로 글이 빠질 일도 없음)
- 프로젝트 전체에 `join fetch` 는 **0건**. 다른 레포지토리도 N:1 연관은 `@EntityGraph`, 목록은 DTO.

### 3. OSIV off 가 이 선택의 또 다른 이유

```
① 요청 도착
② Controller 진입
③   └ Service 메서드 (@Transactional 시작)
④       └ findById(id) → Post 조회, author 는 프록시
⑤   └ Service 반환 (@Transactional 종료)   ← OSIV off: 영속성 컨텍스트도 여기서 닫힘
⑥ Controller 가 model 에 담고 뷰 이름 반환
⑦ Thymeleaf 렌더링 → ${post.author.nickname}   ← 프록시 초기화 시도
⑧ 응답
```

- OSIV off → 영속성 컨텍스트 수명 = **트랜잭션 수명**. 컨트롤러도 이미 트랜잭션 밖.
- ⑦에서 닫힌 영속성 컨텍스트로 프록시 초기화 → **`LazyInitializationException`** (Lazy 프록시 + Initialization).
- 그래서 화면 데이터는 **트랜잭션 안에서 다 채워서** 내보내야 한다 → 목록 DTO, 상세 `@EntityGraph`. N+1 해결과 OSIV 대응을 **한 번에** 해결한 선택.

### 4. 면접 답변 (study-board 부분)

> "study-board 는 OSIV 를 꺼 두어서 화면에 쓸 데이터를 트랜잭션 안에서 다 채워야 했습니다. 목록은 페이징이 있는데 첨부파일 같은 1:N 컬렉션을 조인하면 행이 불어나 페이지가 깨지기 때문에, 컬렉션은 `size()` 로 개수만 받는 DTO 조회를 썼습니다. 상세는 페이징이 없는 단건 조회라 컬렉션을 함께 가져와도 안전해서 `@EntityGraph` 로 작성자와 파일을 한 번에 로딩했습니다."

## 오답·헷갈린 점

- **전**: study-board 목록은 **join fetch** → **후**: 실제는 **`select new PostSummary` DTO 조회**. 프로젝트에 join fetch 0건. ⚠️ 면접에서 내 프로젝트 기술을 잘못 말하면 치명적 — 코드로 재확인할 것.
- **전**: "select new DTO 로 조회하면 파일 개수만큼 게시글이 늘어나서 EntityGraph 를 썼다" → **후**: 원인과 해결이 **뒤바뀜**. 행을 불리는 건 **컬렉션 조인**이고, 이를 피하려고 목록에서 **DTO + `size()`** 를 씀.
- **전**: "조인된 다른 엔티티의 개수에 따라 목록이 늘어난다" → **후**: **1:N 컬렉션**만 불어남. N:1(작성자) 조인은 안전.
- **전**: 상세가 괜찮은 이유 = "모든 파일을 보여줘야 해서 / 중복 제거하면 돼서" → **후**: 결정적 이유는 **페이징 없는 단건 조회**라 `limit` 에 잘릴 일이 없음.
- **전**: OSIV off 에서 영속성 컨텍스트가 닫히는 시점 = "렌더링이 끝날 때" → "컨트롤러 반환 시" → **후**: **트랜잭션 종료(서비스 반환, ⑤)**. 렌더링 끝까지 열려 있는 건 OSIV **on** 일 때.
- **전**: 렌더링은 서비스 종료 "전" → **후**: **후**.
- **전**: 예외 이름 `LazyLenderException` → **후**: **`LazyInitializationException`**.
- ⚠️ 8/25 OSIV 세션에서 직접 끈 설정인데 "언제 닫히나"를 OSIV on 기준으로 답함 → 복습 우선순위 높음.

## Q&A

- **Q. study-board 목록/상세는 무엇을 썼나?** A. 목록 `select new PostSummary`, 상세 `@EntityGraph({"author","files"})`.
- **Q. 목록에서 files 를 fetch join 하면?** A. 행이 파일 수만큼 불어나 limit 이 깨짐 → Hibernate 가 메모리 페이징 + HHH90003004 경고.
- **Q. 목록은 파일 개수를 어떻게 받나?** A. `size(p.files)` → count 서브쿼리.
- **Q. 상세는 왜 컬렉션을 같이 가져와도 되나?** A. 페이징 없는 단건 조회.
- **Q. OSIV off 에서 LAZY 프록시를 템플릿에서 읽으면?** A. 트랜잭션 종료와 함께 영속성 컨텍스트가 닫혀 `LazyInitializationException`.

## 다음 학습

- `show-sql` 로 `size(p.files)` 가 만드는 SQL 직접 확인
- 컬렉션 두 개를 동시에 fetch join 할 때의 `MultipleBagFetchException` (List → Set, 또는 batch size)
- 컬렉션 + 페이징이 꼭 필요할 때: 부모만 페이징 조회 + `default_batch_fetch_size` 로 컬렉션 초기화
- 연결: [[open-in-view-osiv]], [[n-plus-1-problem]], [[2026-10-14-n-plus-one-definition-solutions]], [[2026-10-13-why-lazy-default-fetch-join]]
