---
date: 2026-10-19
type: raw
author: dongju
tags: [생성자 주입, constructor injection, 필드 주입, field injection, 의존성 주입, DI, dependency injection, Autowired, RequiredArgsConstructor, Lombok, final, 불변성, immutability, 리플렉션, reflection, 불완전한 객체, 단위 테스트, unit test, Mock, InjectMocks, NullPointerException, NPE, 컴파일 에러, 순환 참조, circular dependency, circular reference, BeanCurrentlyInCreationException, 조기 참조, early reference, allow-circular-references, Spring Boot 2.6, StackOverflowError, 단일 책임 원칙, SRP, study-board, 면접 답변]
topic: 생성자 주입과 필드 주입의 차이 — 주입 시점(생성 중 vs 생성 후)이 final·테스트·순환 참조·SRP 차이의 뿌리다
summary: 생성자 주입은 빈 인스턴스를 만드는 순간 생성자 인자로 의존성을 받고, 필드 주입은 기본 생성자로 빈 객체를 먼저 만든 뒤 리플렉션으로 private 필드를 채운다. 이 시점 차이 때문에 필드 주입은 final 을 못 쓰고, 스프링 없이 new 하면 의존성이 null 인 불완전한 객체가 되어 런타임 NPE 로만 문제를 알게 되며, 순환 참조도 조기 참조로 통과시켜 운영 중 StackOverflowError 로 숨긴다. 생성자 주입은 이 문제들을 컴파일·기동 시점에 드러내고, 의존성이 많으면 긴 생성자로 SRP 위반을 드러낸다.
source: session
distilled: true
distilled_at: 2026-10-19
---

## 배운 개념

### 1. 핵심은 "언제 주입되느냐"

스프링이 빈을 만드는 순서(단순화):

```
① 인스턴스 생성 (생성자 호출)  →  ② 의존성 채우기  →  ③ 초기화(@PostConstruct 등)
```

- **생성자 주입**: 의존성이 **①에서 생성자 인자로** 들어온다. 의존성 없이는 객체가 아예 만들어지지 않는다.
- **필드 주입**: ①에서 **기본 생성자**로 빈 객체를 만들고, **②에서 리플렉션**으로 `private` 필드를 열어 값을 꽂는다. ①~② 사이에는 **의존성이 `null`인 불완전한 객체**가 존재한다.

```java
// 생성자 주입 (Lombok @RequiredArgsConstructor 는 이 생성자를 대신 써줄 뿐, DI 자체가 아니다)
@Service
public class OrderService {
    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
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

### 2. final 과 불변성

- `final` 필드는 **생성자가 끝나기 전까지** 값이 정해져야 한다.
- 필드 주입은 생성자가 **끝난 뒤(②)** 값을 넣으므로 `final`과 양립 불가 → 주입 후 재할당을 막을 수 없다.
- 생성자 주입은 `final` 가능 → 주입 후 **불변성 보장**.

### 3. 스프링 없는 단위 테스트

```java
OrderService service = new OrderService();  // 필드 주입 버전
service.order(1L);                           // orderRepository == null → NullPointerException
```

| | 필드 주입 | 생성자 주입 |
|---|---|---|
| `new OrderService()` | 컴파일됨, 의존성 `null` | **컴파일 에러** (기본 생성자 없음) |
| 문제 발견 시점 | **런타임** (`NPE`) | **컴파일 타임** |
| Mock 넣는 법 | 리플렉션·`@InjectMocks` 등 **프레임워크 도움** 필요 | `new OrderService(mockRepository)` **순수 자바** |

### 4. 순환 참조 (A → B → A)

- **생성자 주입**: A 생성자를 부르려면 **완성된 B 객체**가 필요하고, B 생성자는 완성된 A가 필요 → 스프링이 생성 중인 빈을 다시 요청받아 **`BeanCurrentlyInCreationException`**, 애플리케이션이 **기동 시점에 실패**.
- **필드 주입**: 기본 생성자로 ①을 의존성 없이 끝낼 수 있으므로 통과됐다.

```
1. AService 를 기본 생성자로 생성 (필드는 null)
2. 덜 만들어진 A 의 참조를 미리 꺼내둠  ← 조기 참조(early reference)
3. A 의 필드를 채우려고 B 생성 → B 의 필드에 덜 만들어진 A 주입
4. B 완성 → A 의 필드에 B 주입 → A 완성
```

- 앱이 뜨는 게 문제인 이유: `A.process()` → `B.handle()` → `A.process()` … **무한 재귀 호출 → `StackOverflowError`**를 **운영 중에야** 발견한다. 순환 참조 자체가 두 클래스의 책임이 엉켰다는 설계 신호인데, 필드 주입은 그걸 숨긴다.
- ⚠️ **Spring Boot 2.6+** 는 순환 참조가 기본 금지(`spring.main.allow-circular-references=false`)라 필드 주입이어도 기동 시 실패한다. `true` 로 풀어야 예전처럼 동작.

### 5. 트레이드오프: 의존성이 8~10개면?

- 생성자 주입은 매개변수가 길어져 **눈에 보인다**. 필드 주입은 `@Autowired` 몇 줄 뒤에 숨는다.
- 의존성 과다 = **단일 책임 원칙(SRP) 위반 신호**. 따라서 긴 생성자는 생성자 주입의 단점이라기보다 **설계 문제를 드러내는 장점**. 해법은 생성자를 줄이는 게 아니라 **클래스를 쪼개는 것**.

### 한 줄 정리

생성자 주입의 장점은 **"문제를 더 빨리, 더 확실하게 드러낸다"**로 묶인다 — 누락은 컴파일 타임에, 순환 참조는 기동 시점에, 책임 과다는 코드를 읽는 시점에.

## 오답·헷갈린 점

- **전**: 생성자 주입은 "클래스 자체에 어노테이션을 달아서" 설정한다 → **후**: 그건 Lombok `@RequiredArgsConstructor` 가 **생성자 코드를 생성**해주는 것. 주입은 스프링이 그 생성자를 호출하며 일어난다.
- **전**: "객체가 만들어진 시점에 주입한다" (두 방식 구분 없음) → **후**: 생성자 주입은 **생성 중(①)**, 필드 주입은 **생성 후(②) 리플렉션**. 이 구분이 final 불가·불완전 객체·순환 참조 차이의 뿌리.
- **전**: 필드 주입의 테스트 문제는 "테스트 중간 로직에서 변조할 가능성" → **후**: 진짜 문제는 **스프링 없이 생성하면 의존성이 `null`** 이라 테스트 시작부터 막히고, Mock 주입에 프레임워크가 필요하다는 것.
- **전**: 필드 주입 순환 참조가 통과하는 이유는 "스프링이 주입해주기 때문" → **후**: **기본 생성자로 ①을 먼저 끝내고, 덜 만들어진 객체를 조기 참조로 넘겨주기 때문**. (생성자 주입도 스프링이 주입하므로 차이를 설명 못 하는 답)
- **전**: "무한루프" → **후**: **무한 재귀 호출 → `StackOverflowError`**.
- **전**: 의존성이 늘면 "책임이 무거워져 수정 리소스가 많이 든다" → **후**: 원인의 이름은 **SRP 위반**. 그리고 긴 생성자는 단점이 아니라 그 위반을 **드러내는 신호**.
- ⚠️ 반복 테마: **"에러가 발생한다"를 3회** 사용 (`NPE` / 컴파일 에러 / `BeanCurrentlyInCreationException` 을 이름으로 말하지 못함). 10/18 "현상은 맞게 짚는데 용어로 환원하지 못함"과 같은 패턴. 면접에선 **무슨 에러가, 언제(컴파일/기동/런타임)** 까지 말할 것.

## Q&A

- **Q. 필드 주입과 생성자 주입은 주입 시점이 어떻게 다른가?** A. 생성자 주입은 인스턴스 생성 시 생성자 인자로, 필드 주입은 기본 생성자로 만든 뒤 리플렉션으로.
- **Q. 필드 주입은 왜 final 을 못 쓰나?** A. final 은 생성자 종료 전에 값이 정해져야 하는데, 필드 주입은 생성자 종료 후에 값을 넣으므로.
- **Q. 스프링 없이 `new OrderService()` 후 `order()` 호출하면?** A. 필드 주입: 의존성 null → `NullPointerException`. 생성자 주입: 기본 생성자가 없어 컴파일 에러.
- **Q. 생성자 주입 테스트에서 Mock 은?** A. `new OrderService(mockRepository)` 로 직접 전달.
- **Q. A↔B 순환 참조를 생성자 주입으로 하면?** A. 서로의 완성된 객체가 필요해 `BeanCurrentlyInCreationException`, 기동 실패.
- **Q. 필드 주입이면?** A. 기본 생성자 + 조기 참조로 기동 성공(Boot 2.6 이전) → 운영 중 무한 재귀 시 `StackOverflowError`. Boot 2.6+ 는 기본 금지.
- **Q. 생성자 주입은 의존성이 많으면 단점 아닌가?** A. 긴 생성자는 SRP 위반을 드러내는 신호. 클래스를 쪼갤 것.

## 다음 학습

- 스프링이 순환 참조를 해결하던 **3단계 캐시**(`singletonObjects` / `earlySingletonObjects` / `singletonFactories`)와 AOP 프록시가 얽힐 때의 문제
- **세터 주입**은 언제 쓰나 (선택적 의존성), `@Autowired(required = false)` / `ObjectProvider`
- 생성자가 하나면 `@Autowired` 생략이 가능한 이유 (Spring 4.3+)
- 연결: [[static-util-vs-spring-bean]], [[2026-10-18-entity-default-constructor-protected]] (리플렉션·기본 생성자), [[2026-10-09-hidden-input-time-testability]] (테스트 용이성)
