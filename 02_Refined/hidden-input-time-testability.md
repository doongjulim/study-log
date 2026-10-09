---
type: refined
slug: hidden-input-time-testability
tags: [숨은입력, hidden-input, 순수함수, pure-function, 결정적, deterministic, 비결정적테스트, flaky-test, LocalDate.now, now, 현재시각, current-time, Clock, Clock.fixed, Clock.systemDefaultZone, 시간의존성, time-dependency, 테스트용이성, testability, D-day, ChronoUnit, 매개변수로드러내기, 경계로밀어내기, functional-core-imperative-shell, 랜덤, random, UUID, static유틸, 의존성주입]
topic: 메서드 안의 LocalDate.now() 같은 숨은 입력은 테스트를 날마다 깨뜨린다 — 매개변수로 드러내거나 Clock 을 주입한다
summary: calcDday(deadline) 가 내부에서 LocalDate.now() 를 부르면 시그니처는 입력 1개처럼 보여도 시스템 시계라는 두 번째 입력을 몰래 읽어, 같은 인자에도 날마다 결과가 바뀌고 테스트가 다음 날 깨진다. 오늘 날짜를 calcDday(today, deadline) 매개변수로 드러내 순수 함수로 만들고 now() 는 서비스(경계) 한 곳에서만 부르거나, Clock 을 빈으로 주입해 테스트에서 Clock.fixed 를 넣는다. 테스트는 now() 를 부르지 않고 고정값을 넣는다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/time/Clock.html
updated: 2026-10-09
---

# 숨은 입력과 테스트 — `now()` 를 밖으로

## 개념 정의

```java
public final class DateUtils {
    public static long calcDday(LocalDate deadline) {
        return ChronoUnit.DAYS.between(LocalDate.now(), deadline);
    }
}

@Test
void 마감_3일_전이면_D3() {
    long dday = DateUtils.calcDday(LocalDate.of(2026, 10, 12));
    assertThat(dday).isEqualTo(3);   // 10/9 엔 통과, 10/10 CI 에선 2 → 실패
}
```

- 시그니처는 입력 1개(`deadline`)처럼 보이지만, **"오늘 날짜"를 시스템 시계에서 몰래 읽는** 두 번째 입력이 있다 = **숨은 입력(Hidden Input)**.
- 같은 인자 → 날마다 다른 결과 → **순수 함수가 아니다** → static 유틸 기준 3 위반 ([[static-util-vs-spring-bean]]).
- 범인은 테스트가 넘긴 고정값(`deadline`)이 아니라 메서드 안의 **`now()`** 다.
- 같은 부류: `Random`, `UUID.randomUUID()`, 시스템 환경변수, 외부 API 응답.

## 작동 원리 — 두 가지 해법

### 방법 1: 숨은 입력을 매개변수로 드러낸다 (static 유지)

```java
public static long calcDday(LocalDate today, LocalDate deadline) {
    return ChronoUnit.DAYS.between(today, deadline);
}
```

```java
// 테스트 — 고정값을 직접 넣는다 (now() 호출 ❌)
@Test
void 마감_3일_전이면_D3() {
    long dday = DateUtils.calcDday(LocalDate.of(2026, 10, 9), LocalDate.of(2026, 10, 12));
    assertThat(dday).isEqualTo(3);   // 언제 돌려도 통과
}

// 서비스 — 실제 "오늘"을 아는 유일한 곳
long dday = DateUtils.calcDday(LocalDate.now(clock), study.getDeadline());
```

바뀌는 값은 **바깥 경계(서비스) 한 곳**에서만 읽고, 계산 로직은 순수 함수로 남긴다 (Functional Core, Imperative Shell).

### 방법 2: `Clock` 을 빈으로 주입한다

시간도 외부 의존성이다 (기준 1).

```java
@Bean Clock clock() { return Clock.system(ZoneId.of("Asia/Seoul")); }        // 운영

Clock fixed = Clock.fixed(Instant.parse("2026-10-09T00:00:00Z"), ZoneId.of("Asia/Seoul"));   // 테스트
LocalDate today = LocalDate.now(clock);
```

## 트레이드오프 / 한계

- 방법 1은 가장 단순하지만, 호출부가 매번 `today` 를 넘겨야 한다. 호출부가 많으면 결국 그 호출부들이 `Clock` 을 주입받는 구조(방법 2)와 결합된다.
- 방법 2는 테스트에서 시간을 **흐르게** 하기 어렵다 (고정 시계). 만료 로직 테스트엔 `Clock.offset` 이나 직접 만든 가변 시계가 필요하다.
- 메서드 안에 `LocalDate.of(2026, 10, 9)` 같은 값을 박는 건 해결이 아니다 — 운영이 영원히 그 날짜 기준이 된다.

> [!WARNING]
> **오답 코너**
> - **"테스트가 깨지는 건 변수값이 고정돼 있어서"** — 고정값(테스트가 넘긴 deadline)은 원하는 것이다. 범인은 메서드 안에서 매일 바뀌는 **`LocalDate.now()`** 다.
> - **"now() 대신 고정값으로 바꾸면 된다"** — 메서드 안에 박으면 운영이 깨진다. **매개변수로 드러내** 밖에서 넣게 한다.
> - **"now() 는 서비스와 테스트에서 호출한다"** — 테스트는 `LocalDate.of(...)` **고정값**을 넣는다. `now()` 는 서비스(경계) 한 곳에서만 부른다.

## 복습 체크

- [ ] `calcDday(deadline)` 테스트가 다음 날 깨지는 이유를 "숨은 입력" 으로 설명할 수 있는가?
- [ ] static 을 유지하며 고치는 새 시그니처와, 그때 테스트·서비스 코드는?
- [ ] `now()` 는 어디서만 불러야 하는가?
- [ ] `Clock` 주입 방식에서 운영/테스트 각각 어떤 Clock 을 넣는가?
- [ ] 시간 외에 숨은 입력이 되는 것 두 가지를 들 수 있는가?

## 관련

[[static-util-vs-spring-bean]] · [[optional-orelse-vs-orelseget]] · [[deterministic-hash-for-lookup]]
