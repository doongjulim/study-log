---
type: refined
slug: open-in-view-osiv
tags: [open-in-view, OSIV, osiv, spring.jpa.open-in-view, Open-Session-In-View, 영속성컨텍스트생명주기, OpenEntityManagerInViewInterceptor, 커넥션점유, 커넥션풀고갈, pool-exhaustion, HikariCP, connection-timeout, 트랜잭션밖변경, 의도치않은UPDATE, JPA, Hibernate, Spring-Boot]
topic: spring.jpa.open-in-view 가 영속성 컨텍스트를 뷰까지 열어두는 대가와, 끄는 것이 왜 기본 선택인가
summary: OSIV 는 영속성 컨텍스트를 뷰 렌더링 완료까지 열어두어 뷰 계층 지연 로딩을 가능하게 하지만, 그 대가로 DB 커넥션을 요청 끝까지 점유하고 트랜잭션 밖 엔티티 변경이 후속 트랜잭션의 flush 에 딸려 나간다.
contributors: [dongju]
source_refs:
  - https://vladmihalcea.com/the-open-session-in-view-anti-pattern/
updated: 2026-08-25
---

# Open Session In View (`spring.jpa.open-in-view`)

## 개념 정의

**영속성 컨텍스트(Hibernate `Session` / JPA `EntityManager`)의 생명주기를 어디까지 늘릴지** 결정하는 Spring Boot 설정.

| 설정 | 영속성 컨텍스트 생존 범위 |
|---|---|
| `true` (Spring Boot 기본값) | 요청 시작 ~ **뷰 렌더링 완료**까지 |
| `false` | `@Transactional` 시작 ~ 종료까지 |

옵션 이름 그대로 *"view 에서도 (컨텍스트를) 열어둔다"* 는 뜻이다.

**핵심 뉘앙스: `true` 여도 트랜잭션 자체는 서비스 계층에서 시작하고 끝난다.**
늘어나는 것은 트랜잭션이 아니라 영속성 컨텍스트뿐이다. 그래서 `true` 에는
**"트랜잭션은 이미 커밋됐는데 컨텍스트만 살아있는 구간"** 이 생기고,
OSIV 의 모든 편의와 모든 사고가 정확히 이 구간에서 발생한다.

## 작동 원리

### 왜 기본값이 `true` 인가 — 지연 로딩 편의

`false` 상태에서 컨트롤러·뷰가 지연 로딩 프록시를 건드리면 `LazyInitializationException` 이 터진다
([[lazy-loading-proxy]]). `true` 면 컨텍스트가 살아있으므로 서비스 계층에서 미리 fetch join / DTO 변환을
하지 않아도 뷰에서 연관 엔티티를 자유롭게 탐색할 수 있다. **개발 편의성**이 존재 이유다.

### 커넥션 점유 구간

```
① 요청 도착
② 컨트롤러 진입
③ 서비스 @Transactional 시작 → 조회        ← 커넥션 획득 (lazy acquisition, 첫 쿼리 시점)
④ 서비스 @Transactional 커밋/종료           ← 트랜잭션 종료. 그래도 커넥션은 안 놓는다
⑤ 컨트롤러 복귀, 외부 API 호출 (3초)        ← DB 쿼리 0건인데 커넥션 점유 중
⑥ 뷰 렌더링 (지연 로딩 발생)
⑦ 응답 완료                                 ← EntityManager close → 커넥션 반납
```

- **점유 구간 = ③(첫 쿼리) ~ ⑦(요청 완료)**
- ④에서 커넥션을 놓지 못하는 이유: **⑥에서 언제 지연 로딩이 튀어나올지 모르기 때문.**
  → **"영속성 컨텍스트가 살아있다" ≒ "커넥션도 붙잡고 있다"**
- 반납이 ⑥이 아니라 ⑦인 이유: ⑥은 아직 지연 로딩이 일어나는 중이다. 실제로는
  `OpenEntityManagerInViewInterceptor` 가 **뷰 렌더링까지 끝난 뒤 `afterCompletion`** 에서 닫는다.

### 트랜잭션 밖 변경이 DB 로 새는 경로

```java
@GetMapping("/posts/{id}")
public String view(@PathVariable Long id, Model model) {
    Post post = postService.findPost(id);   // 영속 상태로 반환
    post.setTitle("컨트롤러에서 수정");        // ← 트랜잭션 밖
    someOtherService.doSomething();          // ← 여기서 새 @Transactional 커밋
}
```

1. 변경 감지는 값을 바꾸는 순간이 아니라 **flush 순간**에 동작한다 ([[dirty-checking]]).
2. `true` 라 요청 내내 컨텍스트가 **하나** → `post` 는 여전히 **영속(managed)** = 스냅샷 비교 대상.
3. 무관한 서비스의 커밋이 flush 를 일으키고, flush 는 **컨텍스트 안의 모든 영속 엔티티**를 훑는다.
4. → **아무도 의도하지 않은 `UPDATE post SET title=...`**

무서운 이유는 **변경한 코드(컨트롤러)와 DB 에 반영시킨 코드(무관한 서비스)가 분리**되어 원인 추적이 어렵다는 점.

## 트레이드오프 / 한계

**끄면 얻는 것**

- 커넥션을 트랜잭션 종료와 함께 반납 → 풀 회전율 상승
- 엔티티가 트랜잭션 종료 시 **준영속(detached)** 이 되어 의도치 않은 UPDATE 원천 봉쇄
- 지연 로딩이 계층 밖으로 새지 않아 쿼리 발생 지점이 서비스 계층으로 고정됨

**끄면 치러야 할 대가**

| 계열 | 방법 |
|---|---|
| 트랜잭션 안에서 미리 채우기 | `join fetch`, `@EntityGraph`, `default_batch_fetch_size` ([[n-plus-1-problem]]) |
| 애초에 엔티티를 안 넘기기 | DTO 직접 조회(projection), 서비스에서 DTO 변환 후 반환 |

**켜둔 채 방치했을 때의 장애 시나리오 (풀 10, 초당 20 요청)**

1. 11번째 요청부터 커넥션 대기(blocking)
2. 앞 요청들은 ⑤의 외부 API 를 기다리느라 커넥션을 안 놓음
3. 대기 시간이 `connection-timeout`(HikariCP 기본 30초) 초과
4. `SQLTransientConnectionException: Connection is not available, request timed out`
5. **커넥션 풀 고갈(pool exhaustion)** → 같은 풀을 공유하는 **전체 서비스로 장애 전파**

> 트래픽이 적은 어드민·백오피스라면 편의를 택해 켜둘 수도 있다. 판단 기준은
> "이 애플리케이션이 커넥션 회전율을 걱정할 만큼의 동시 요청을 받는가"이다.

> [!WARNING]
> **오답 코너**
> - **`true`/`false` 의미 반전** — `true` 가 뷰까지, `false` 가 트랜잭션 범위다. 옵션 이름이 근거.
> - **"OSIV 는 무결성을 위해 존재한다"** — 반대다. 무결성 문제는 **켰을 때 생기는 부작용**이고,
>   존재 이유는 뷰 계층 지연 로딩 편의다.
> - **"끄면 메모리를 아낀다"** — 메모리는 부수적. 진짜는 **DB 커넥션의 조기 반납**. 커넥션은 풀 크기가
>   고정된 자원이라 고갈되면 서비스 전체가 죽는다.
> - **"커넥션 반납은 뷰 렌더링 시점"** — 렌더링 **완료 후**(`afterCompletion`). 렌더링 중에도 지연 로딩이 있다.
> - **"커넥션 오버플로우"** — 풀은 넘치지 않는다. 정확한 용어는 **고갈(exhaustion) + 대기(blocking) + 타임아웃**.
> - **"트랜잭션 밖 변경은 DB 에 반영되지 않는다"** — `true` 면 **반영될 수 있다**. 영속 상태 유지 +
>   후속 트랜잭션의 flush 조합. 인과의 주어는 "뒤에 열린 남의 트랜잭션"이다.

## 복습 체크

- [ ] `true`/`false` 각각의 영속성 컨텍스트 생존 범위를 말할 수 있는가?
- [ ] `true` 여도 트랜잭션 범위는 변하지 않는다는 점을 설명할 수 있는가?
- [ ] 커넥션 획득 시점(③)과 반납 시점(⑦)을, 그 이유와 함께 말할 수 있는가?
- [ ] 트랜잭션이 끝났는데도 커넥션을 못 놓는 이유를 한 문장으로 답할 수 있는가?
- [ ] 커넥션 풀 고갈이 왜 해당 API 하나가 아니라 전체 서비스를 죽이는지 설명할 수 있는가?
- [ ] 트랜잭션 밖 엔티티 변경이 DB 로 새는 4단계 인과 사슬을 순서대로 말할 수 있는가?
- [ ] 끈 뒤에 해야 하는 두 계열의 처방을 각각 예시와 함께 댈 수 있는가?

## 관련

[[lazy-loading-proxy]] · [[n-plus-1-problem]] · [[dirty-checking]] · [[db-pagination]]
