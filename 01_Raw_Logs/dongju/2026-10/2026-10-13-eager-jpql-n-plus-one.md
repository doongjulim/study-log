---
date: 2026-10-13
type: raw
author: dongju
tags: [즉시 로딩, EAGER, FetchType.EAGER, 지연 로딩, LAZY, FetchType.LAZY, N+1, n plus 1, JPQL, findAll, em.find, 1차 캐시, first-level cache, ManyToOne, fetch 설정, 추가 쿼리, 연관관계, association]
topic: EAGER 인데 왜 조인 1번이 아니라 N+1 인가 — JPQL 은 글자 그대로 번역되고 fetch 설정은 조회 이후에 반영된다
summary: em.find 는 EAGER 설정을 보고 조인 SQL 1번을 만들지만, findAll 같은 JPQL 은 fetch 설정과 무관하게 그대로 SQL 로 번역된 뒤 결과 엔티티를 보고 EAGER 연관을 추가 조회해서 N+1 이 생긴다. 추가 쿼리 수는 서로 다른 연관 엔티티 수이며, 이미 조회한 엔티티는 1차 캐시에서 꺼낸다.
source: session
distilled: true
distilled_at: 2026-10-14
---

## 배운 개념

### 1. 상황

`Post` → `Member` 가 `@ManyToOne`. 게시글 목록 화면은 제목·작성일만 쓰고 작성자는 안 쓴다.

```java
postRepository.findAll();   // 게시글 20개
```

### 2. 같은 EAGER 인데 결과가 다르다

| 호출 | 실제 쿼리 |
|---|---|
| `em.find(Post.class, 1L)` | `post` + `member` **조인 1번** |
| `postRepository.findAll()` | **N+1** |

### 3. 왜 JPQL 은 N+1 인가 — 시간 순서

```
① JPQL "select p from Post p" → SQL 로 글자 그대로 번역
   → select * from post            (member 조인 없음)
② 결과 Post 20개가 영속성 컨텍스트에 올라옴
③ 매핑 정보를 보니 member 가 EAGER → 즉시 채워야 함
   → member 마다 select * from member where id = ? 추가 실행
```

> **JPQL 은 fetch 설정과 무관하게 그대로 SQL 로 번역되고, fetch 설정은 조회 이후 시점에 반영된다.**

### 4. 추가 쿼리는 정확히 몇 번?

- 게시글 20개, 작성자 3명 → 추가 쿼리 **3번** (총 1+3).
- 1번 글에서 `Member(id=1)` 을 조회하면 1차 캐시에 올라감 → 5번 글이 같은 `member_id=1` 을 가리키면 DB 대신 **1차 캐시**에서 꺼냄.
- 즉 추가 쿼리 수 = **서로 다른 연관 엔티티 수**.

### 5. EAGER 의 두 가지 문제

1. 화면에서 쓰지도 않는 연관 엔티티까지 항상 조회한다.
2. JPQL 조회 시 조인이 아니라 N+1 로 가져온다 — 예측하기 어려운 SQL.

## 오답·헷갈린 점

- **전**: "JPA 가 SQL 을 만들 때 EAGER 설정을 참고한다" → **후**: 참고한다면 조인 1번으로 끝났어야 함(모순). JPQL 은 **그대로 번역**되고 EAGER 는 **조회 이후**에 추가 쿼리로 반영된다.
- **전**: 작성자 3명일 때 추가 쿼리 3번인 이유 = "ID 값이 없는 경우는 조회할 게 없어서" → **후**: 그건 `member_id` 가 null 인 경우 얘기. 실제 이유는 이미 조회한 엔티티를 **1차 캐시**에서 꺼내기 때문.
- ⚠️ 반복 테마: JPA 동작 "시점"(번역 시점 vs 조회 이후) 구분이 흐림 — 지난 flush 세션의 "플러시 시점" 혼동과 같은 계열.

## Q&A

- **Q. EAGER 상태에서 쓰지 않는 작성자 정보 때문에 생기는 문제는?** A. 안 쓰는 member 까지 조회 + JPQL 이면 N+1.
- **Q. em.find 는 조인 1번인데 findAll 은 왜 N+1?** A. findAll 은 JPQL 이라 fetch 설정 없이 그대로 번역 → 조회 후 EAGER 연관을 추가 조회.
- **Q. 게시글 20개, 작성자 3명이면 추가 쿼리는?** A. 3번. 같은 작성자는 1차 캐시에서 꺼냄.

## 다음 학습

- `show-sql` 로 `em.find` vs `findAll` (EAGER) 쿼리 로그 직접 비교
- `@OneToMany` 컬렉션 EAGER 에서 N+1 이 더 커지는 이유
- 연결: [[n-plus-1-problem]], [[persistence-context]], [[2026-10-13-jpa-proxy-structure-equals]], [[2026-10-13-why-lazy-default-fetch-join]]
