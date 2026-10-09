---
type: refined
slug: static-util-vs-spring-bean
tags: [static, 정적메서드, static-method, 유틸클래스, utility-class, util, 스프링빈, spring-bean, Component, 빈vsstatic, static-vs-bean, 의존성주입, DI, dependency-injection, Value, Autowired, application.yml, 설정값, PasswordEncoder, BCrypt, Argon2, DelegatingPasswordEncoder, 구현교체, 테스트대역, test-double, mock, mockStatic, 정적바인딩, static-binding, invokestatic, method-hiding, 순수함수, pure-function, 공유가변상태, shared-mutable-state, SimpleDateFormat, DateTimeFormatter, 판단기준, 면접답변]
topic: static 유틸로 뺄지 스프링 빈으로 만들지의 기준 — 의존성·교체 필요·순수성·공유 가변 상태 4가지
summary: "여러 곳에서 호출된다"는 기준이 아니다(Math.max 도 static). 외부 의존성(설정값·다른 빈·시간)이 필요하면 빈이고, 구현 교체·공존·테스트 대역이 필요하면 빈이다. static 필드는 클래스에 붙어 JVM 이 로딩하므로 스프링이 주입할 수 없고, static 메서드는 invokestatic 정적 바인딩이라 다형성이 없다. 같은 입력에 항상 같은 출력을 내는 순수 계산이고 static 필드에 가변 상태가 없을 때만 static 으로 둔다.
contributors: [dongju]
source_refs:
  - https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html
  - https://docs.spring.io/spring-security/reference/features/authentication/password-storage.html
updated: 2026-10-09
---

# static 유틸 vs 스프링 빈 — 판단 기준

## 개념 정의

| # | 질문 | Yes면 |
|---|---|---|
| 1 | **외부 의존성**(설정값, 다른 빈, **시간**)이 필요한가? | **빈** |
| 2 | 구현 **교체·공존·테스트 대역**이 필요한가? | **빈** |
| 3 | **같은 입력 → 항상 같은 출력**(순수 함수)인가? | static 가능 |
| 4 | static 필드에 **가변 상태**가 있는가? | static 금지 (불변 객체로) |

> 한 문장: **"의존성도, 교체 필요도, 숨은 입력도, 공유 가변 상태도 없는 순수 계산만 static 으로 뺀다."**

**"여러 곳에서 호출된다"는 기준이 아니다.** `Math.max`, `StringUtils.hasText`, `Objects.requireNonNull` 은 수백 곳에서 호출되지만 전부 static 이다. static 도 어디서든 부를 수 있다.

## 작동 원리 — 왜 이 기준인가

### 기준 1: static 에는 주입이 안 된다

- 스프링이 `@Value`/`@Autowired` 로 값을 넣는 대상은 **스프링이 만든 객체(빈)의 필드**다.
- `static` 필드는 객체가 아니라 **클래스**에 붙고, 클래스는 **JVM 이 로딩**한다 → 스프링이 관여할 수 없다.
- enum 상수에 `@Autowired` 가 안 되는 것과 같은 이유 ([[enum-constant-specific-polymorphism]]).
- 의존성 종류: 설정값(yml), 다른 빈(Repository), **현재 시각**(`Clock`) ([[hidden-input-time-testability]]), 외부 API.

### 기준 2: static 은 다형성이 없다

```java
// (A) static 유틸
String hash = PasswordUtils.encode(raw);

// (B) 빈 + 인터페이스
private final PasswordEncoder passwordEncoder;
String hash = passwordEncoder.encode(raw);
```

BCrypt → Argon2 전환 시:

| 상황 | (A) static | (B) 빈 + 인터페이스 |
|---|---|---|
| 단순 교체 | `encode` 몸통 1곳 (호출부는 그대로) | 빈 등록 1곳 |
| **두 구현 공존** (기존 회원 BCrypt 검증 + 신규 Argon2 저장) | 몸통 안 if 분기 | `DelegatingPasswordEncoder` 처럼 **실행 시점 구현 선택** |
| **테스트 교체** (BCrypt 는 일부러 느림) | 바꿀 방법이 사실상 없음 (`mockStatic` 은 최후의 수단) | 가짜 구현·Mock 주입 |

static 메서드는 오버라이딩되지 않고(hiding), 호출은 `invokestatic` 으로 컴파일 시점에 고정되는 **정적 바인딩**이다 ([[overriding-vs-overloading-binding]]).

### 기준 3: 숨은 입력이 있으면 순수 함수가 아니다

```java
public static long calcDday(LocalDate deadline) {
    return ChronoUnit.DAYS.between(LocalDate.now(), deadline);   // now() = 숨은 입력
}
```

같은 인자에도 날마다 결과가 바뀐다 → 오늘 날짜를 매개변수로 드러내면 다시 static 순수 함수가 된다 ([[hidden-input-time-testability]]).

### 기준 4: static 필드는 JVM 전체 스레드가 공유한다

```java
private static final SimpleDateFormat FORMAT = new SimpleDateFormat("yyyy-MM-dd");   // ❌ 내부 Calendar 가변
private static final DateTimeFormatter FORMAT = DateTimeFormatter.ofPattern("yyyy-MM-dd");   // ✅ 불변
```

`final` 은 참조만 고정할 뿐 내부 상태 변경은 못 막는다 → 동시 요청에서 경쟁 조건 ([[race-condition-lost-update]], [[shallow-immutability-defensive-copy]]).

### 사례 판정

| 기능 | 결론 | 근거 |
|---|---|---|
| `toSlug(title)` | static (한 곳에서만 쓰면 그 클래스의 private 메서드) | 순수 함수, 의존성 없음 |
| `encode(raw)` | **빈** (`PasswordEncoder`) | yml 설정(1) + 공존·테스트 교체(2) |
| `calcDday(deadline)` + `now()` | ❌ → `calcDday(today, deadline)` | 숨은 입력(3) |
| `static final SimpleDateFormat` | ❌ → `DateTimeFormatter` | 공유 가변 상태(4) |

## 트레이드오프 / 한계

- 빈은 컨테이너 등록·주입 보일러플레이트가 있고, 호출하려면 그 빈을 주입받아야 한다. 순수 계산까지 빈으로 만들면 의존성 그래프만 복잡해진다.
- static 유틸은 관례상 `final` 클래스 + `private` 생성자로 인스턴스화를 막는다.
- 스프링 싱글턴 빈도 모든 요청 스레드가 공유한다 — 빈이라고 기준 4에서 자유롭지 않다 ([[jvm-stack-and-heap]]).

### 면접 답변 뼈대

> "기준은 네 가지입니다. 외부 설정이나 다른 빈, 현재 시각 같은 **의존성이 있으면 빈**으로 만들고, 구현을 **교체하거나 테스트에서 대역으로 바꿔야 하면 빈**으로 만듭니다. static 은 정적 바인딩이라 다형성이 없기 때문입니다.
> 반대로 **같은 입력에 항상 같은 출력을 내는 순수 계산**이고 상태가 없으면 static 유틸로 둡니다. 예를 들어 D-day 계산은 처음에 내부에서 `LocalDate.now()` 를 불러 테스트가 날마다 깨질 수 있었는데, 오늘 날짜를 매개변수로 받게 바꿔서 순수 함수로 만들고 `now()` 는 서비스에서만 부르도록 했습니다.
> 그리고 static 필드에는 `SimpleDateFormat` 같은 가변 객체를 두지 않고, `DateTimeFormatter` 같은 불변 객체만 둡니다."

> [!WARNING]
> **오답 코너**
> - **"여러 곳에서 호출되니까 빈으로 만든다"** — `Math.max` 도 수백 곳에서 호출되는 static 이다. 기준은 호출 횟수가 아니라 **의존성·교체 필요·순수성·상태**다.
> - **"static 이면 구현 교체 시 호출부를 전부 고쳐야 한다"** — 단순 교체는 몸통 1곳이면 된다. static 이 막히는 건 **두 구현 공존**과 **테스트 교체**다.
> - **"yml 값을 static 으로 못 받는다" (결론만)** — 이유까지: 스프링은 **빈 객체의 필드**에만 주입하고, static 은 JVM 이 로딩한 클래스에 붙는다.

## 복습 체크

- [ ] 판단 기준 4가지를 질문 형태로 말할 수 있는가?
- [ ] "여러 곳에서 쓰인다" 가 기준이 아닌 이유를 예로 반박할 수 있는가?
- [ ] static 필드에 `@Value` 가 안 되는 이유는?
- [ ] 비밀번호 해시가 static 이 아니라 빈이어야 하는 이유 두 가지(설정, 공존·테스트)는?
- [ ] `toSlug` / `encode` / `calcDday` / `SimpleDateFormat` 을 각각 판정하고 근거를 댈 수 있는가?
- [ ] 면접 답변 뼈대에 내 프로젝트 장면을 붙여 말할 수 있는가?

## 관련

[[hidden-input-time-testability]] · [[overriding-vs-overloading-binding]] · [[enum-constant-specific-polymorphism]] · [[race-condition-lost-update]] · [[shallow-immutability-defensive-copy]] · [[secret-hashing-algorithm-choice]] · [[jvm-stack-and-heap]]
