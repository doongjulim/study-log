---
type: refined
slug: jpa-dto-projection
tags: [DTO조회, DTO-projection, 프로젝션, projection, select-new, 생성자표현식, constructor-expression, Object[], 스칼라조회, scalar, JPQL, record-DTO, 필요한컬럼만, 영속성컨텍스트비관리, 변경감지없음, 프록시없음, 지연로딩없음, OSIV, open-in-view, LazyInitializationException, 페이징, size(), 컬렉션개수, 서브쿼리, N+1, 목록조회, 화면전용]
topic: JPQL select new 로 화면에 필요한 값만 DTO 로 바로 조회하기 — 엔티티 조회와의 차이, Object[] 와의 차이, 목록 + 페이징 + OSIV off 에서 강한 이유
summary: select 절에 엔티티 대신 new 패키지.클래스(값...) 를 적으면 필요한 컬럼만 쿼리 1번으로 DTO 에 담긴다. 값만 나열하면 Object[] 로 와서 직접 캐스팅해야 한다. DTO 는 영속성 컨텍스트가 관리하지 않아 프록시·지연 로딩·변경 감지가 모두 없으므로, OSIV 를 꺼도 뷰에서 터질 것이 없고, 컬렉션은 size() 로 개수만 받아 페이징을 지킬 수 있다. 대신 수정에는 쓸 수 없다.
contributors: [dongju]
source_refs:
  - https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html
updated: 2026-10-14
---

# DTO 조회 (select new)

## 개념 정의

JPQL `select` 절에 엔티티 대신 **DTO 생성자 호출**을 적어, 화면에 필요한 값만 쿼리 한 번으로 받는 방식.

```java
public record PostListItem(Long id, String title, String authorNickname, LocalDateTime createdAt) {}

@Query("select new com.example.board.post.dto.PostListItem(p.id, p.title, m.nickname, p.createdAt) " +
       "from Post p join p.member m")
List<PostListItem> findListItems();
```

목록 화면에 필요한 건 작성자 **엔티티 전체**가 아니라 **닉네임 하나**뿐이라는 관찰에서 출발한다.

## 작동 원리

### `select new` vs 값 나열

| 쓰는 법 | 행마다 받는 타입 |
|---|---|
| `select p` | 엔티티 `Post` (영속 상태) |
| `select p.id, p.title, m.nickname` | **`Object[]`** — 직접 꺼내고 캐스팅 |
| `select new 패키지.PostListItem(p.id, p.title, m.nickname)` | **DTO** — 생성자로 바로 |

- `new` 뒤에는 **패키지를 포함한 전체 클래스명**을 적는다(JPQL 은 import 를 모른다).
- 인자 순서·타입이 DTO 생성자와 일치해야 한다. record 의 정규 생성자가 잘 맞는다 ([[java-record]]).

### 영속성 컨텍스트가 관리하지 않는다

DTO 는 엔티티가 아니므로 1차 캐시에 올라가지 않는다. 그래서

- **프록시가 없다** → 지연 로딩도 없다 → `LazyInitializationException` 이 날 수 없다.
- **스냅샷이 없다** → 변경 감지 비용이 없다 ([[dirty-checking]]).
- 연관 값은 쿼리에서 이미 값으로 꺼냈으므로 N+1 이 생길 자리가 없다 ([[n-plus-1-problem]]).

### 컬렉션은 개수만 — `size()`

```java
select new ...PostSummary(p.id, p.title, p.author.nickname, size(p.files), ...) from Post p
```

- `size(p.files)` 는 SQL 에서 `(select count(*) from attached_file where post_id = p.id)` 같은 **서브쿼리**가 된다.
- 글 하나당 숫자 하나 → 1:N 조인처럼 **행이 불어나지 않아** DB `limit` 페이징이 그대로 유지된다.
- `p.author.nickname` 같은 N:1 경로는 암묵적 조인이 되며, 이것도 행을 불리지 않는다.

## 트레이드오프 / 한계

| 장점 | 대가 |
|---|---|
| 필요한 컬럼만, 쿼리 1번 | **수정에 쓸 수 없다** — 영속 엔티티가 아니라 변경 감지가 없다 |
| OSIV off 에서도 뷰에서 안전 | 화면이 바뀌면 쿼리·DTO 를 함께 고쳐야 한다(화면에 결합) |
| 컬렉션 개수를 페이징 안전하게 | 문자열 JPQL 이라 오타가 런타임에 드러난다(구동 시 검증되긴 함) |

**언제 쓰나** — 목록처럼 **읽기 전용 + 페이징 + 일부 컬럼**인 화면. 상세처럼 단건에서 엔티티 그래프를 다 써야 하면
`@EntityGraph` 가 더 자연스럽다.

### 사례 — 게시판 목록

OSIV 를 끈 게시판에서 목록은 `select new PostSummary(..., size(p.files), ...)` + `Page<PostSummary>`,
상세는 `@EntityGraph(attributePaths = {"author", "files"})` 로 나눴다. 목록은 페이징이 있어 컬렉션을 조인하면
페이지가 깨지고, 화면에 필요한 건 파일 **개수**뿐이었기 때문이다 ([[open-in-view-osiv]]).

> [!WARNING]
> **오답 코너**
> - **"`m.nickname` 처럼 필요한 값만 select 하면 DTO 로 받는다"** — 값만 나열하면 **`Object[]`** 다. `select new 전체클래스명(...)` 이 필요하다.
> - **"DTO 조회를 쓰면 파일 개수만큼 글이 늘어난다"** — 원인과 해결이 뒤바뀐 것. 행을 불리는 건 **컬렉션 조인**이고,
>   DTO + `size()` 는 그걸 **피하는** 방법이다.
> - **"DTO 로 받은 뒤 setter 로 바꾸면 DB 에 반영된다"** — 영속성 컨텍스트가 관리하지 않으므로 반영되지 않는다.

## 복습 체크

- [ ] `select p`, 값 나열, `select new` 가 각각 어떤 타입으로 돌아오는가?
- [ ] `select new` 에 전체 클래스명을 써야 하는 이유는?
- [ ] DTO 조회에서 `LazyInitializationException` 과 N+1 이 생길 수 없는 이유를 영속성 컨텍스트로 설명할 수 있는가?
- [ ] `size(p.files)` 가 SQL 로 어떻게 바뀌며, 왜 페이징을 깨지 않는가?
- [ ] DTO 조회를 쓰지 말아야 할 상황은?

## 관련

[[n-plus-1-problem]] · [[lazy-loading-proxy]] · [[open-in-view-osiv]] · [[dirty-checking]] · [[db-pagination]] · [[java-record]]
