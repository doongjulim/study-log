---
type: refined
slug: spring-data-repository-proxy
tags: [동적프록시, dynamic-proxy, JDK동적프록시, 프록시빈, RepositoryFactoryBean, SimpleJpaRepository, 리포지토리구현체, 인터페이스구현, 싱글턴빈, singleton, 구동시점, startup, AOP, 프록시패턴, Transactional프록시, 부가기능, Spring-Data-JPA]
topic: 구현 클래스가 없는 리포지토리 인터페이스가 실행되는 이유 — 구동 시점에 만들어지는 동적 프록시 싱글턴 빈
summary: Spring Data JPA 는 앱 구동 시 리포지토리 인터페이스를 찾아 이를 구현하는 프록시를 런타임 생성해 싱글턴 빈으로 등록한다. 기본 메서드는 SimpleJpaRepository 에 위임하고, @Transactional 이 동작하는 것도 같은 프록시 가로채기 원리다.
contributors: [dongju]
source_refs:
  - https://docs.spring.io/spring-data/jpa/reference/repositories/core-concepts.html
updated: 2026-09-26
---

# 리포지토리 동적 프록시

## 개념 정의

```java
public interface PostRepository extends JpaRepository<Post, Long> { }
```

`implements` 한 클래스를 어디에도 쓰지 않았는데 `postRepository.findAll()` 이 동작한다.
**Spring Data JPA 가 구동 시점에 이 인터페이스를 구현하는 객체를 런타임에 만들어 빈으로 등록**하기 때문이다.

## 작동 원리

**앱 구동 시 딱 한 번** 일어나는 일:

1. 컴포넌트 스캔이 `Repository` 를 상속한 인터페이스를 발견
2. `RepositoryFactoryBean` 이 그 인터페이스를 구현하는 **동적 프록시 객체**를 생성 (JDK 동적 프록시)
3. 그 프록시를 **싱글턴 빈**으로 컨테이너에 등록 → 필요한 곳에 주입
4. 이때 파생 쿼리 메서드 이름도 함께 파싱된다 ([[derived-query-method]])

**요청마다 만드는 것이 아니다.** 구동 시 1회 생성된 싱글턴이 모든 요청을 처리한다.

호출이 들어오면 프록시가 분기한다:

| 호출 대상 | 처리 |
|---|---|
| `findAll`, `save`, `findById` 등 기본 메서드 | **`SimpleJpaRepository`** 구현체에 위임 |
| `findByTitleContaining` 등 파생 메서드 | 구동 시 파싱해 둔 쿼리를 실행 |
| `@Query` 붙은 메서드 | 선언된 JPQL/네이티브 쿼리 실행 |
| 사용자 정의 구현(`XxxRepositoryImpl`) | 해당 구현체로 위임 |

### 같은 원리의 다른 얼굴 — `@Transactional`

`@Transactional` 이 트랜잭션을 자동으로 여닫는 것도 **완전히 같은 메커니즘**이다.

```
호출 → [프록시] 트랜잭션 시작 → 실제 객체의 메서드 실행 → 커밋(또는 롤백) → 반환
```

프록시가 **호출을 가로채(intercept) 부가 기능을 앞뒤로 덧붙이는 것** — 이것이 Spring AOP 의 본질이다.
리포지토리 프록시와 트랜잭션 프록시는 "프록시가 가로채 부가 기능을 수행한다"는 하나의 원리를 공유한다.

"커밋(또는 롤백)"을 가르는 것도 프록시다. 메서드 밖으로 **나가는 예외의 타입**을 보고 결정하며,
기본값은 `RuntimeException`·`Error` 만 롤백하고 Checked 예외는 커밋한다. 안에서 catch 로 삼킨 예외는 프록시가 보지 못해 커밋된다
([[transactional-rollback-rules]]).

## 트레이드오프 / 한계

- **프록시를 거치지 않으면 부가 기능도 없다.** 같은 클래스 안에서 `this.someTransactionalMethod()` 로
  자기 호출(self-invocation)하면 프록시를 통과하지 않아 **트랜잭션이 걸리지 않는다.** 대표적 함정.
- **`private`/`final` 메서드에는 적용되지 않는다** — 가로챌 방법이 없다.
- 인터페이스 기반 JDK 동적 프록시는 **인터페이스 타입으로만** 주입받을 수 있다.
  (일반 스프링 빈은 인터페이스가 없으면 CGLIB 프록시를 쓴다 — Spring Boot 의 기본은 CGLIB 프록시다.)
- 실행 스택이 한 겹 늘어 스택 트레이스가 길어지고 디버깅이 다소 어려워진다.

> [!WARNING]
> **오답 코너**
> - **"리포지토리 구현체는 요청마다 생성된다"** — 구동 시 **1회 생성되는 싱글턴 프록시 빈**이다.
>   요청마다 빌리는 것은 **DB 커넥션**이고, 이는 커넥션 풀이라는 **별개 계층**의 자원이다.
> - **"필터가 가로채서 트랜잭션을 연다"** — 필터는 서블릿 계층에서 **HTTP 요청**을 가로채는 별개 개념이다.
>   메서드 호출을 가로채는 것은 **프록시**.
> - **"같은 클래스 안에서 호출해도 `@Transactional` 이 걸린다"** — self-invocation 은 프록시를 거치지 않는다.

## 복습 체크

- [ ] 구현 클래스 없는 인터페이스가 실행되는 이유를 한 문장으로 말할 수 있는가?
- [ ] 프록시가 만들어지는 시점과 개수(스코프)를 정확히 답할 수 있는가?
- [ ] 기본 메서드는 어디로 위임되는가?
- [ ] `@Transactional` 이 리포지토리 프록시와 같은 원리라는 것을 설명할 수 있는가?
- [ ] self-invocation 시 트랜잭션이 안 걸리는 이유는?
- [ ] "요청마다 커넥션을 만든다"와 "구동 시 프록시 1회 생성"을 구분해 설명할 수 있는가?

## 관련

[[derived-query-method]] · [[handler-method-argument-resolver]] · [[open-in-view-osiv]] · [[spring-application-events]] · [[transactional-rollback-rules]]
