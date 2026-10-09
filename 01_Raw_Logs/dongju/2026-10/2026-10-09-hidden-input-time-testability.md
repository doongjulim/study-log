---
date: 2026-10-09
type: raw
author: dongju
tags: [숨은 입력, hidden input, 순수 함수, pure function, 결정적, deterministic, 비결정적 테스트, flaky test, LocalDate.now, 현재 시각, current time, Clock, Clock.fixed, Clock.systemDefaultZone, 시간 의존성, time dependency, 테스트 용이성, testability, D-day, ChronoUnit, 매개변수로 드러내기, 경계로 밀어내기, functional core imperative shell, static 유틸, 의존성 주입]
topic: 메서드 안의 LocalDate.now() 는 숨은 입력이다 — 매개변수로 드러내 순수 함수로 만들거나 Clock 을 주입해 테스트를 결정적으로 만든다
summary: calcDday(deadline) 가 내부에서 LocalDate.now() 를 부르면 시그니처는 입력 1개처럼 보여도 시스템 시계라는 두 번째 입력을 몰래 읽어, 같은 인자에도 날마다 결과가 바뀌고 테스트가 다음 날 깨진다. 고치는 법은 오늘 날짜를 calcDday(today, deadline) 매개변수로 드러내 static 순수 함수로 남기고 now() 는 서비스(경계) 한 곳에서만 부르거나, Clock 을 빈으로 주입받아 테스트에서 Clock.fixed 를 넣는 것이다. 테스트는 now() 를 부르지 않고 고정값을 넣는다.
source: session
distilled: false
---

## 배운 개념

### 1. 문제 코드

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

```java
DateUtils.calcDday(LocalDate.of(2026, 10, 12))      // ← 테스트가 넘긴 값: 고정 (문제 아님)
ChronoUnit.DAYS.between(LocalDate.now(), deadline)  // ← LocalDate.now(): 날마다 바뀜 (범인)
```

- 시그니처는 입력 1개(`deadline`)처럼 보이지만, 실제로는 **"오늘 날짜"를 시스템 시계에서 몰래 읽는** 두 번째 입력이 있다 = **숨은 입력(Hidden Input)**.
- 같은 인자 → 날마다 다른 결과 → **순수 함수가 아니다** → static 유틸 기준 3 위반 ([[static-util-vs-spring-bean]]).

### 2. 방법 1 — 숨은 입력을 매개변수로 드러내기 (static 유지)

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
    assertThat(dday).isEqualTo(3);   // 내일 돌려도, 1년 뒤 돌려도 항상 통과
}

// 서비스 — 실제 "오늘"을 아는 유일한 곳
long dday = DateUtils.calcDday(LocalDate.now(clock), study.getDeadline());
```

- 바뀌는 값(`now()`)은 **바깥 경계(서비스) 한 곳**에서만 읽고, 계산 로직은 순수 함수로 남긴다.
- 메서드 안에 `LocalDate.of(2026, 10, 9)` 를 박는 건 해결이 아니다 → 운영 서버가 영원히 10/9 기준.

### 3. 방법 2 — Clock 을 빈으로 주입

- 시간도 **외부 의존성**(기준 1) → 주입 대상.
- 운영: `Clock.systemDefaultZone()` (또는 `Clock.system(ZoneId.of("Asia/Seoul"))`)
- 테스트: `Clock.fixed(Instant.parse("2026-10-09T00:00:00Z"), ZoneId.of("Asia/Seoul"))`
- 사용: `LocalDate.now(clock)`

## 오답·헷갈린 점

- **전**: 테스트 실패 원인 = "**변수값이 고정**되어 있기 때문" → **후**: 고정값(테스트가 넘긴 deadline)은 문제가 아니라 원하는 것. 범인은 메서드 안의 **`LocalDate.now()`(숨은 입력)**. 결과(내일 실패) 예측은 정답.
- **전**: 해결 = "**now() 가 아닌 고정값으로 바꿈**" (위치 불명) → **후**: 메서드 안에 박으면 운영이 깨진다. **매개변수(`today`)로 드러내** 밖에서 넣게 한다.
- **전**: 바꾼 뒤 `now()` 는 "**서비스와 테스트**에서 호출" → **후**: 테스트는 `LocalDate.of(...)` **고정값**. `now()` 는 서비스(경계) 한 곳에서만 (또는 Clock 주입).
- **정답 맞힘**: 새 시그니처의 첫 매개변수 `today`.

## Q&A

- **Q. 이 테스트를 내일 돌리면?** A. 실패(2 반환). 원인은 메서드 내부 `LocalDate.now()`.
- **Q. static 을 유지하며 고치려면?** A. `calcDday(LocalDate today, LocalDate deadline)` — 숨은 입력을 매개변수로.
- **Q. 그럼 `now()` 는 누가 부르나?** A. 서비스(경계). 테스트는 고정값.
- **Q. 빈으로 고친다면?** A. `Clock` 주입, 테스트에서 `Clock.fixed`.

## 다음 학습

- `Clock` 빈 등록과 `@TestConfiguration` 으로 고정 시계 교체
- 함수형 코어, 명령형 셸 (Functional Core, Imperative Shell)
- 랜덤(`UUID.randomUUID()`, `Random`)도 같은 숨은 입력 — 주입·시드 고정
