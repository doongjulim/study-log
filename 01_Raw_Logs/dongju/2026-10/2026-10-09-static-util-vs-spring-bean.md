---
date: 2026-10-09
type: raw
author: dongju
tags: [static, 정적 메서드, static method, 유틸 클래스, utility class, util, 스프링 빈, spring bean, Component, 빈 vs static, static vs bean, 의존성 주입, DI, dependency injection, Value, Autowired, application.yml, 설정값, PasswordEncoder, BCrypt, Argon2, DelegatingPasswordEncoder, 구현 교체, 테스트 대역, test double, mock, 정적 바인딩, static binding, invokestatic, method hiding, 오버라이딩 불가, 순수 함수, pure function, 판단 기준, 면접 답변]
topic: 스프링 빈이 아니라 static 유틸 클래스로 빼는 기준 — 의존성·교체 필요·순수성·공유 가변 상태 4가지
summary: "여러 곳에서 호출된다"는 기준이 아니다(Math.max 도 static). 외부 의존성(설정값·다른 빈·시간)이 필요하면 빈, 구현 교체·공존·테스트 대역이 필요하면 빈이다. static 필드는 클래스에 붙어 JVM 이 로딩하므로 스프링이 @Value/@Autowired 를 넣을 수 없고, static 메서드는 오버라이딩이 안 되는 정적 바인딩(invokestatic)이라 다형성이 없다. 같은 입력에 항상 같은 출력을 내는 순수 계산이고 공유 가변 상태가 없을 때만 static 으로 둔다.
source: session
distilled: true
distilled_at: 2026-10-09
---

## 배운 개념

### 1. 출발 예시 — 세 기능을 static / 빈 / 내부 메서드로

```java
// ① 게시글 제목을 URL용 문자열로 변환
//    "스프링 JPA 정리!" → "스프링-jpa-정리"
toSlug(String title)

// ② 회원 비밀번호를 해시
//    BCrypt 사용, 강도(strength)는 application.yml 에서 설정
encode(String rawPassword)

// ③ 스터디 마감일까지 남은 일수(D-day) 계산
//    LocalDate.now() 와 deadline 의 차이
calcDday(LocalDate deadline)
```

| 기능 | 결론 | 근거 |
|---|---|---|
| ① `toSlug` | static (한 곳에서만 쓰면 그 클래스의 private 메서드) | 순수 함수, 의존성 없음 |
| ② `encode` | **빈** (`PasswordEncoder`) | yml 설정(기준 1) + 구현 공존·테스트 교체(기준 2) |
| ③ `calcDday` + `now()` | ❌ 그대로는 안 됨 | 숨은 입력 → [[hidden-input-time-testability]] |

- ① 을 "한 곳에서만 쓰면 굳이 밖으로 안 뺀다(private 메서드)" 는 판단은 타당.

### 2. "여러 곳에서 호출된다" 는 기준이 아니다

```java
Math.max(a, b);                 // static
StringUtils.hasText(str);       // static (스프링 제공)
Objects.requireNonNull(obj);    // static
```

수백 곳에서 호출되지만 전부 static. static 도 어디서든 부를 수 있다.

### 3. 기준 1 — 외부 의존성이 필요하면 빈

- 스프링이 `@Value`/`@Autowired` 로 값을 넣는 대상은 **스프링이 만든 객체(빈)의 필드**.
- `static` 필드는 객체가 아니라 **클래스**에 붙고, 클래스는 **JVM 이 로딩** → 스프링이 관여 못 함.
- 같은 날 배운 "enum 상수에 `@Autowired` 불가" 와 같은 이유 ([[enum-constant-specific-polymorphism]]).
- 의존성 종류: 설정값(yml), 다른 빈(Repository 등), **시간**(`Clock`), 외부 API.

### 4. 기준 2 — 구현 교체·공존·테스트 대역이 필요하면 빈

```java
// (A) static 유틸
String hash = PasswordUtils.encode(raw);          // 10곳에서 이렇게 호출

// (B) 빈 + 인터페이스
private final PasswordEncoder passwordEncoder;    // 10곳에서 주입받아
String hash = passwordEncoder.encode(raw);
```

BCrypt → Argon2 교체 시:

| 상황 | (A) static | (B) 빈 + 인터페이스 |
|---|---|---|
| 단순 교체 | `PasswordUtils.encode` 몸통 1곳 (호출부 10곳은 그대로) | 빈 등록 1곳 |
| **두 구현 공존** (기존 회원 BCrypt 검증 + 신규 Argon2 저장) | 몸통 안에 if 분기 | `DelegatingPasswordEncoder` 처럼 **실행 시점에 구현 선택**(다형성) |
| **테스트 교체** (BCrypt 는 일부러 느림) | 바꿀 방법 없음 | 가짜 구현·Mock 주입 |

- **static 메서드는 오버라이딩이 안 된다.** 자식에 같은 static 메서드를 만들면 **hiding** 일 뿐, 호출은 `invokestatic PasswordUtils.encode` 로 컴파일 시점에 고정 → **정적 바인딩**, 다형성 없음 ([[overriding-vs-overloading-binding]]).

### 5. 기준 3·4 — 순수 함수, 공유 가변 상태 없음

- 기준 3: **같은 입력 → 항상 같은 출력**이면 static OK. 시간·랜덤·외부 상태 같은 숨은 입력이 있으면 X → [[hidden-input-time-testability]]
- 기준 4: static 필드에 **가변 상태** 금지. `final` 이어도 내부가 가변이면 위험 → [[static-shared-mutable-state]]

### 6. 판단표

| # | 질문 | Yes면 |
|---|---|---|
| 1 | **외부 의존성**(설정값, 다른 빈, 시간)이 필요한가? | **빈** |
| 2 | 구현 **교체·공존·테스트 대역**이 필요한가? | **빈** |
| 3 | **같은 입력 → 항상 같은 출력**(순수 함수)인가? | static 가능 |
| 4 | static 필드에 **가변 상태**가 있는가? | static 금지 (불변 객체로) |

> 한 문장: **"의존성도, 교체 필요도, 숨은 입력도, 공유 가변 상태도 없는 순수 계산만 static 으로 뺀다."**

### 7. 면접 답변 뼈대

> "기준은 네 가지입니다. 외부 설정이나 다른 빈, 현재 시각 같은 **의존성이 있으면 빈**으로 만들고, 구현을 **교체하거나 테스트에서 대역으로 바꿔야 하면 빈**으로 만듭니다. static 은 정적 바인딩이라 다형성이 없기 때문입니다.
> 반대로 **같은 입력에 항상 같은 출력을 내는 순수 계산**이고 상태가 없으면 static 유틸로 둡니다. 예를 들어 D-day 계산은 처음에 내부에서 `LocalDate.now()` 를 불러 테스트가 날마다 깨질 수 있었는데, 오늘 날짜를 매개변수로 받게 바꿔서 순수 함수로 만들고 `now()` 는 서비스에서만 부르도록 했습니다.
> 그리고 static 필드에는 `SimpleDateFormat` 같은 가변 객체를 두지 않고, `DateTimeFormatter` 같은 불변 객체만 둡니다."

(TODO: study-board 에서 실제로 static 으로 뺀 클래스명 + "처음엔 이렇게 → 이런 이유로 바꿈" 장면 붙이기)

## 오답·헷갈린 점

- **전**: `encode` 는 "**여러 곳에서 호출될 수 있기 때문에**" 빈 → **후**: `Math.max` 도 수백 곳에서 호출되는 static. 진짜 이유는 **외부 설정 의존**(기준 1) + **구현 교체·공존·테스트 대역**(기준 2).
- **전**: yml 값을 static 으로 못 받는다 (결론만) → **후**: 스프링은 **빈 객체의 필드**에만 주입. static 은 클래스에 붙고 클래스는 JVM 이 로딩.
- **전**: 해시 교체 시 **11군데** 수정 → **후**: (질문 설계상 함정) 단순 교체는 static 도 몸통 1곳. static 이 막히는 건 **두 구현 공존**과 **테스트 교체**.
- **정답 맞힘**: static 메서드는 오버라이딩 안 됨.

## Q&A

- **Q. static 유틸로 뺀 기준이 뭔가?** A. 판단표 4가지 + 면접 답변 뼈대.
- **Q. static 메서드 안에서 `@Value` 를 쓸 수 있나?** A. 없다. 주입 대상은 빈 객체의 필드.
- **Q. static 메서드를 오버라이딩할 수 있나?** A. 없다. hiding 만 되고 호출은 invokestatic 으로 컴파일 시점 고정.
- **Q. 비밀번호 해시는 왜 빈인가?** A. 설정 의존 + 알고리즘 공존(DelegatingPasswordEncoder) + 테스트 교체.

## 다음 학습

- 유틸 클래스 관례: `final` 클래스 + `private` 생성자 (인스턴스화 방지)
- Mockito `mockStatic` 은 왜 최후의 수단인가
- `DelegatingPasswordEncoder` 의 `{bcrypt}` 접두어 방식과 해시 마이그레이션
