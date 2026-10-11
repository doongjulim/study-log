---
date: 2026-10-11
type: raw
author: dongju
tags: [Java 17, 자바 17, LTS, 장기 지원, long-term support, Spring Boot 3, Spring Framework 6, Java 17 baseline, jakarta, javax, record, 레코드, switch 식, switch expression, 화살표 case, 텍스트 블록, text block, instanceof 패턴 매칭, pattern matching for instanceof, Stream.toList, sealed, 봉인 클래스, preview, 미리보기 기능, 버전 선택, 면접 답변, study-board, exhaustiveness, 망라성 검사, default 없는 switch]
topic: 왜 Java 17 인가, 그리고 "17 에서 쓴 기능"은 사실 14~16 에서 정식화된 기능을 17 LTS 가 묶은 것 — study-board 실제 사용처와 default 없는 switch 식의 누락 검사
summary: Java 17 을 고른 이유는 Spring Boot 3.x(Spring 6)의 최소 요건이 Java 17 이고 그중 LTS 가 17 이기 때문이다. study-board 에서 쓴 record·switch 식·텍스트 블록·instanceof 패턴 매칭·Stream.toList 는 14~16 에서 정식화된 기능이고 17 에서 정식화된 대표 기능은 sealed(미사용)이므로, "17 의 신기능"이 아니라 "11 이후 정식화된 기능을 LTS 로 처음 묶은 버전이 17"이라고 말해야 정확하다. SearchType 분기는 리포지토리(빈)를 호출하므로 enum 상수별 구현 대신 서비스의 default 없는 switch 식으로 두어 상수 누락을 컴파일 에러로 잡는다.
source: session
distilled: false
---

## 배운 개념

### 1. 왜 Java 17 인가

```groovy
plugins {
    id 'org.springframework.boot' version '3.3.5'
}
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}
```

- **Spring Boot 3.x (Spring Framework 6) 의 최소 요건 = Java 17.** 이때 `javax.*` → `jakarta.*` 패키지 전환도 함께 일어났다.
- 자바는 6개월마다 출시되지만 몇 년씩 보안 패치를 받는 건 **LTS(11, 17, 21)** 뿐. 운영 서비스는 LTS 를 고른다.
- → "Boot 3 를 쓰려면 17 이상이 강제됐고, 그중 LTS 가 17 이었다" 가 가장 정직한 답.

### 2. "17 에서 쓴 기능" ≠ "17 의 신기능"

study-board 실제 사용처 (코드 확인 기준):

| 기능 | 정식 도입 | study-board 사용처 |
|---|---|---|
| switch 식 (`case X ->`) | **14** | `PostService` 의 `SearchType` 분기, `PostController` 정렬·탭, `NotificationType.allowedBy` |
| 텍스트 블록 (`"""`) | **15** | `PostRepository` 의 `@Query` |
| `record` | **16** | DTO·값 객체 약 40곳 (`PostSummary`, `DashboardView`, `GoalProgress` …) |
| `instanceof` 패턴 매칭 | **16** | `SocialLoginProviders` |
| `Stream.toList()` | **16** | 21곳 |
| `sealed` 클래스 | **17** | 미사용 ← 17 에서 정식화된 대표 기능 |

- 12·13 은 대부분 **미리보기(preview)** 단계.
- 면접 표현: **"11 이후 정식화된 기능들을 LTS 로 처음 한데 묶은 버전이 17 이라서 쓸 수 있게 된 것"**. "17 의 신기능" 이라고 하면 "record 는 16 아닌가요?" 로 짚힌다.

### 3. default 없는 switch 식 = 컴파일러 누락 검사

```java
public enum SearchType {
    TITLE("제목"), TITLE_CONTENT("제목+내용"), WRITER("작성자");
    ...
}

// PostService
return switch (type) {
    case TITLE         -> postRepository.findSummariesByTitle(keyword, category, pageable);
    case TITLE_CONTENT -> postRepository.findSummariesByTitleOrContent(keyword, category, pageable);
    case WRITER        -> postRepository.findSummariesByAuthorNickname(keyword, category, pageable);
};   // ← default 가 없다
```

- `SearchType` 에 `CONTENT` 만 추가하고 분기를 안 고치면:

```
error: the switch expression does not cover all possible input values
```

- 상수별 구현이 주던 "빠뜨리면 컴파일 에러" 안전망을 switch 식으로도 얻는다 → **`default` 를 일부러 안 쓰는 게 포인트.**
- 각 분기가 `postRepository`(스프링 빈)를 호출 → enum 상수별 구현으로 옮길 수 없다 (enum 상수는 JVM 이 만든 static 싱글턴, 주입 불가 — [[enum-constant-specific-polymorphism]]). enum 은 "무엇을"(타입·라벨), 서비스는 "어떻게".
- 대조: `NotificationType.allowedBy(NotificationPreference preference)` 는 enum **안에서** `switch (this)` — 빈 대신 필요한 것을 **매개변수**로 받으니 enum 안에 둘 수 있었고, switch 식이라 똑같이 누락 검사를 받는다.

```java
public boolean allowedBy(NotificationPreference preference) {
    return switch (this) {
        case REMINDER -> preference.isReminderEnabled();
        case PLAN_SHARED -> preference.isPlanSharedEnabled();
        ...
    };
}
```

### 4. 면접 답변 뼈대 (Java 17 파트)

> "Spring Boot 3 를 쓰려면 Java 17 이상이 필수였고, 그중 LTS 가 17 이라 선택했습니다. 실제로는 14부터 16 사이에 정식화된 record, switch 식, 텍스트 블록, 패턴 매칭을 썼습니다. 예를 들어 검색 타입 분기는 `default` 없는 switch 식으로 짜서, 검색 타입이 추가됐는데 분기를 빠뜨리면 컴파일 에러가 나게 했습니다."

## 오답·헷갈린 점

- 이번 파트는 큰 오개념 없이 정답. 기록할 주의점:
  - "17 에서 쓴 기능" 을 "17 의 신기능" 으로 말하는 함정 — 스스로 "12~17 사이 기능을 쓸 수 있는 LTS 가 17" 이라고 정리함. 정확한 정식 버전은 14·15·16, 17 신규 대표는 sealed.
  - switch 식 누락 시 컴파일 에러 / repository(빈) 호출이라 enum 으로 못 옮김 — 둘 다 정답 (10/9 enum 학습이 실제 코드 판단과 연결됨).
- 세션 시작 시 일반론 질문에 답하지 않고 질문을 반복 → 실제 프로젝트 코드를 열어 근거 기반으로 진행하니 답변이 구체화됨.

## Q&A

- **Q. 왜 Java 17 인가? 11 이나 21 은?** A. Spring Boot 3 최소 요건이 17 + LTS.
- **Q. record·switch 식·텍스트 블록은 17 신기능인가?** A. 아니다. 16·14·15 정식. 17 이 그것들을 LTS 로 묶었다. 17 신규 대표는 sealed.
- **Q. SearchType 분기를 enum 상수별 구현이 아니라 switch 로 둔 이유?** A. 분기마다 리포지토리(빈) 호출 → enum 에 주입 불가. 대신 default 없는 switch 식으로 누락 검사 확보.

## 다음 학습

- `sealed` 클래스 + switch 패턴 매칭(Java 21)으로 할 수 있는 것
- 가상 스레드(Java 21)와 Spring Boot 3.2+ `spring.threads.virtual.enabled`
- `javax` → `jakarta` 전환이 왜 일어났나 (Java EE → Jakarta EE 이관)
