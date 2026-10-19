---
type: refined
slug: constructor-vs-field-injection
tags: [생성자주입, constructor-injection, 필드주입, field-injection, 세터주입, setter-injection, 의존성주입, DI, dependency-injection, Autowired, RequiredArgsConstructor, Lombok, final, 불변성, immutability, 리플렉션, reflection, 기본생성자, 불완전한객체, 빈생성순서, 단위테스트, unit-test, Mock, InjectMocks, NullPointerException, NPE, 컴파일에러, 순환참조, circular-dependency, circular-reference, BeanCurrentlyInCreationException, 조기참조, early-reference, allow-circular-references, Spring-Boot-2.6, StackOverflowError, 단일책임원칙, SRP, 면접답변]
topic: 생성자 주입과 필드 주입의 차이 — 주입 시점(생성 중 vs 생성 후)이 final·테스트·순환 참조·SRP 차이의 뿌리
summary: 생성자 주입은 빈을 만드는 순간 생성자 인자로 의존성을 받고, 필드 주입은 기본 생성자로 빈 객체를 먼저 만든 뒤 리플렉션으로 private 필드를 채운다. 이 시점 차이 때문에 필드 주입은 final 을 못 쓰고, 스프링 없이 new 하면 의존성이 null 인 불완전한 객체가 되어 런타임 NPE 로만 문제를 알며, 순환 참조도 조기 참조로 통과시켜 운영 중 StackOverflowError 로 숨긴다. 생성자 주입은 같은 문제를 컴파일·기동 시점에 드러내고, 의존성 과다는 긴 생성자로 SRP 위반을 드러낸다.
contributors: [dongju]
source_refs:
  - https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html
  - https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.6-Release-Notes
updated: 2026-10-19
---

# 생성자 주입 vs 필드 주입

## 개념 정의

둘 다 스프링이 의존성을 넣어 주지만 **언제** 넣느냐가 다르다. 스프링이 빈을 만드는 순서(단순화):

```
① 인스턴스 생성 (생성자 호출)  →  ② 의존성 채우기  →  ③ 초기화(@PostConstruct 등)
```

| | 생성자 주입 | 필드 주입 |
|---|---|---|
| 주입 시점 | **①** 생성자 인자로 | **②** 생성 후 리플렉션으로 `private` 필드에 |
| ①~② 사이 상태 | 없음. 의존성 없이는 객체가 안 만들어짐 | 의존성이 `null` 인 **불완전한 객체** 존재 |

```java
// 생성자 주입
@Service
public class OrderService {
    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {   // Spring 4.3+: 생성자 하나면 @Autowired 생략 가능
        this.orderRepository = orderRepository;
    }
}

// 필드 주입
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;
}
```

Lombok `@RequiredArgsConstructor` 는 위 생성자 코드를 **대신 써 줄 뿐** DI 자체가 아니다. 주입은 스프링이 그 생성자를 호출하며 일어난다.

## 작동 원리 — 시점 차이가 만드는 4가지 차이

### 1. final 과 불변성

`final` 필드는 **생성자가 끝나기 전에** 값이 정해져야 한다. 필드 주입은 생성자가 **끝난 뒤(②)** 값을 넣으므로 `final` 과 양립할 수 없다 → 주입 후 재할당을 막을 수 없다. 생성자 주입은 `final` 가능. (단 `final` 은 참조만 고정한다 — [[shallow-immutability-defensive-copy]])

### 2. 스프링 없는 단위 테스트

```java
OrderService service = new OrderService();  // 필드 주입 버전 — 컴파일됨
service.order(1L);                           // orderRepository == null → NullPointerException
```

| | 필드 주입 | 생성자 주입 |
|---|---|---|
| `new OrderService()` | 컴파일됨, 의존성 `null` | **컴파일 에러** (기본 생성자 없음) |
| 누락 발견 시점 | **런타임** (`NPE`) | **컴파일 타임** |
| Mock 넣기 | 리플렉션·`@InjectMocks` 등 프레임워크 필요 | `new OrderService(mockRepository)` 순수 자바 |

### 3. 순환 참조 (A → B → A)

- **생성자 주입**: A 생성자를 부르려면 **완성된 B** 가 필요하고 B 생성자는 완성된 A 가 필요 → 생성 중인 빈을 다시 요청받아 **`BeanCurrentlyInCreationException`**, **기동 시점에 실패**.
- **필드 주입**: 기본 생성자로 ①을 의존성 없이 끝낼 수 있어 통과된다.

```
1. AService 를 기본 생성자로 생성 (필드는 null)
2. 덜 만들어진 A 의 참조를 미리 꺼내 둠  ← 조기 참조(early reference)
3. A 의 필드를 채우려고 B 생성 → B 의 필드에 덜 만들어진 A 주입
4. B 완성 → A 의 필드에 B 주입 → A 완성
```

앱이 뜨는 게 문제다. `A.process()` → `B.handle()` → `A.process()` … **무한 재귀 → `StackOverflowError`** 를 **운영 중에야** 발견한다. 순환 참조는 두 클래스의 책임이 엉켰다는 설계 신호인데 필드 주입은 그걸 숨긴다.

### 4. 의존성 과다

생성자 주입은 의존성이 8~10개면 매개변수가 길어져 **눈에 보인다.** 필드 주입은 `@Autowired` 몇 줄 뒤에 숨는다. 의존성 과다는 **SRP 위반 신호** → 긴 생성자는 단점이 아니라 설계 문제를 **드러내는** 장점이다. 해법은 생성자를 줄이는 게 아니라 **클래스를 쪼개는 것.**

## 트레이드오프 / 한계

- **Spring Boot 2.6+** 는 순환 참조가 기본 금지(`spring.main.allow-circular-references=false`)라 필드 주입이어도 기동 시 실패한다. 순환 참조 측면의 차이는 예전보다 줄었다.
- 선택적 의존성(없어도 동작)은 세터 주입이나 `ObjectProvider` 가 더 자연스럽다.
- 한 줄 정리: 생성자 주입의 장점은 **"문제를 더 빨리, 더 확실하게 드러낸다"** 로 묶인다 — 누락은 컴파일 타임에, 순환 참조는 기동 시점에, 책임 과다는 코드를 읽는 시점에.

> [!WARNING]
> **오답 코너**
> - **"생성자 주입은 클래스에 어노테이션을 달아서 설정한다"** → 그건 Lombok `@RequiredArgsConstructor` 가 **생성자 코드를 생성**하는 것. DI 는 스프링이 그 생성자를 호출하며 일어난다.
> - **"둘 다 객체가 만들어진 시점에 주입한다"** → 생성자 주입은 **생성 중(①)**, 필드 주입은 **생성 후(②) 리플렉션.** 이 구분이 모든 차이의 뿌리.
> - **"필드 주입의 테스트 문제는 테스트 중간에 값을 변조할 수 있다는 것"** → 진짜 문제는 스프링 없이 생성하면 **의존성이 `null`** 이라 테스트 시작부터 막히고 Mock 주입에 프레임워크가 필요하다는 것.
> - **"필드 주입이 순환 참조를 통과하는 건 스프링이 주입해 주기 때문"** → 생성자 주입도 스프링이 주입한다. 이유는 **기본 생성자로 ①을 먼저 끝내고 덜 만들어진 객체를 조기 참조로 넘기기 때문.**
> - **"순환 호출하면 무한루프"** → **무한 재귀 호출 → `StackOverflowError`.**
> - **"에러가 발생한다"** 로 끝내지 말 것 → `NPE`(런타임) / 컴파일 에러 / `BeanCurrentlyInCreationException`(기동) 처럼 **무슨 에러가 언제** 까지.

## 복습 체크

- [ ] 빈 생성 순서 ①②③ 에서 생성자 주입과 필드 주입이 각각 어디서 일어나는지 말할 수 있다
- [ ] 필드 주입이 `final` 을 쓸 수 없는 이유를 생성자 종료 시점으로 설명할 수 있다
- [ ] 스프링 없이 `new` 했을 때 두 방식에서 각각 무슨 일이 (언제) 생기는지 말할 수 있다
- [ ] 생성자 주입 순환 참조 시 나는 예외 이름과 발생 시점을 말할 수 있다
- [ ] 필드 주입이 순환 참조를 통과하던 4단계(조기 참조)를 그릴 수 있고, Boot 2.6 이후 변화를 안다
- [ ] "의존성이 많으면 생성자가 길어진다"는 지적에 SRP 로 반박할 수 있다

## 관련

[[transactional-proxy-self-invocation]] · [[singleton-bean-stateful]] · [[static-util-vs-spring-bean]] · [[shallow-immutability-defensive-copy]] · [[jpa-entity-default-constructor]] · [[hidden-input-time-testability]]
