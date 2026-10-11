---
type: refined
slug: java17-lts-features
tags: [Java17, 자바17, LTS, 장기지원, long-term-support, SpringBoot3, Spring-Boot-3, Spring6, Java17-baseline, jakarta, javax, record, 레코드, switch식, switch-expression, 화살표case, arrow-case, 텍스트블록, text-block, instanceof패턴매칭, pattern-matching-instanceof, Stream.toList, sealed, 봉인클래스, preview, 미리보기기능, 버전선택, exhaustiveness, 망라성검사, default없는switch, 면접답변]
topic: 왜 Java 17 인가 — Spring Boot 3 최소 요건 + LTS, 그리고 "17 에서 쓴 기능"은 14~16 에서 정식화된 기능을 17 LTS 가 묶은 것
summary: Java 17 을 고르는 가장 정직한 이유는 Spring Boot 3.x(Spring 6)의 최소 요건이 17 이고 그중 LTS 가 17 이라는 것이다. record·switch 식·텍스트 블록·instanceof 패턴 매칭·Stream.toList 는 14~16 에서 정식화됐고 17 에서 정식화된 대표 기능은 sealed 클래스이므로, "17 의 신기능"이 아니라 "11 이후 정식화된 기능을 LTS 로 처음 묶은 버전이 17"이라고 말해야 정확하다. default 없는 switch 식은 enum 상수 누락을 컴파일 에러로 잡아 준다.
contributors: [dongju]
source_refs:
  - https://openjdk.org/projects/jdk/17/
  - https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Release-Notes
  - https://openjdk.org/jeps/361
updated: 2026-10-11
---

# Java 17 — 왜 17 이고, 무엇을 썼나

## 개념 정의

### 왜 17 인가

```groovy
plugins { id 'org.springframework.boot' version '3.3.5' }
java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }
```

1. **Spring Boot 3.x (Spring Framework 6) 의 최소 요건이 Java 17.** 이때 `javax.*` → `jakarta.*` 패키지 전환도 같이 일어났다.
2. 자바는 6개월마다 나오지만 몇 년씩 보안 패치를 받는 건 **LTS(11, 17, 21)** 뿐 → 운영 서비스는 LTS.

→ "Boot 3 를 쓰려면 17 이상이 강제됐고, 그중 LTS 가 17 이었다."

### "17 에서 쓴 기능" ≠ "17 의 신기능"

| 기능 | 정식 도입 | 쓰임 예 |
|---|---|---|
| switch 식 (`case X ->`) | **14** | enum 분기, 정렬 조건 분기 |
| 텍스트 블록 (`"""`) | **15** | JPQL `@Query` |
| `record` | **16** | DTO·값 객체 ([[java-record]]) |
| `instanceof` 패턴 매칭 | **16** | 타입 확인 + 캐스팅 한 번에 |
| `Stream.toList()` | **16** | 수정 불가 리스트 반환 ([[stream-vs-for-loop]]) |
| `sealed` 클래스 | **17** | ← 17 에서 정식화된 대표 기능 |

12·13 은 대부분 미리보기(preview) 단계였다.

> 면접 표현: **"11 이후에 정식화된 기능들을 LTS 로 처음 한데 묶은 버전이 17 이라서 쓸 수 있게 된 것."**

## 작동 원리 — default 없는 switch 식의 누락 검사

```java
public enum SearchType { TITLE, TITLE_CONTENT, WRITER }

return switch (type) {
    case TITLE         -> postRepository.findSummariesByTitle(keyword, category, pageable);
    case TITLE_CONTENT -> postRepository.findSummariesByTitleOrContent(keyword, category, pageable);
    case WRITER        -> postRepository.findSummariesByAuthorNickname(keyword, category, pageable);
};   // ← default 가 없다
```

`SearchType` 에 `CONTENT` 만 추가하고 분기를 안 고치면:

```
error: the switch expression does not cover all possible input values
```

- switch **식**은 값을 반환해야 하므로 모든 경우를 다뤄야 한다(exhaustiveness). enum 의 모든 상수를 다뤘다면 `default` 없이도 컴파일된다.
- → enum 상수별 구현이 주던 "빠뜨리면 컴파일 에러" 안전망을 switch 식으로도 얻는다 ([[enum-constant-specific-polymorphism]]). **`default` 를 일부러 쓰지 않는 게 포인트.** `default` 를 넣으면 이 검사가 사라진다.
- 분기가 리포지토리(빈)를 호출하면 enum 안으로 옮길 수 없다 → 서비스의 switch 식이 맞는 자리. 반대로 필요한 값을 **매개변수**로 받으면 enum 안에서 `switch (this)` 로 둘 수 있다.

```java
public boolean allowedBy(NotificationPreference preference) {   // enum 메서드
    return switch (this) {
        case REMINDER -> preference.isReminderEnabled();
        case PLAN_SHARED -> preference.isPlanSharedEnabled();
        ...
    };
}
```

## 트레이드오프 / 한계

- 기존 switch **문**(`case X:` + `break`)은 망라성 검사를 하지 않는다. 검사를 받으려면 **식**으로 써야 한다.
- 21 LTS 는 switch 패턴 매칭, record 패턴, 가상 스레드가 정식화됐다. 17 을 고른 뒤에도 "왜 21 이 아닌가" 에는 시작 시점·생태계·배포 환경을 근거로 답한다.

### 면접 답변 뼈대

> "Spring Boot 3 를 쓰려면 Java 17 이상이 필수였고, 그중 LTS 가 17 이라 선택했습니다. 실제로는 14부터 16 사이에 정식화된 record, switch 식, 텍스트 블록, 패턴 매칭을 썼습니다. 예를 들어 검색 타입 분기는 `default` 없는 switch 식으로 짜서, 검색 타입이 추가됐는데 분기를 빠뜨리면 컴파일 에러가 나게 했습니다."

> [!WARNING]
> **오답 코너**
> - **"record·switch 식·텍스트 블록은 Java 17 의 신기능"** — 각각 16·14·15 에서 정식화됐다. 17 이 그것들을 LTS 로 묶었다. 17 에서 정식화된 대표 기능은 `sealed`.
> - **"그냥 최신 LTS 라서 17"** — 더 정확한 1순위 근거는 **Spring Boot 3 의 최소 요건**이다.
> - **"switch 에는 늘 default 를 넣는 게 안전하다"** — enum 을 다루는 switch 식에서는 `default` 가 **누락 검사를 꺼 버린다.**

## 복습 체크

- [ ] Java 17 을 고른 이유 두 가지를 말할 수 있는가?
- [ ] record, switch 식, 텍스트 블록, instanceof 패턴 매칭의 정식 도입 버전은?
- [ ] 17 에서 정식화된 대표 기능은?
- [ ] default 없는 switch 식이 enum 상수 추가 시 무엇을 해 주는가? default 를 넣으면?
- [ ] 분기를 enum 상수별 구현이 아니라 서비스 switch 식에 둔 이유는?

## 관련

[[java-record]] · [[enum-constant-specific-polymorphism]] · [[stream-vs-for-loop]] · [[jvm-container-heap-sizing]]
