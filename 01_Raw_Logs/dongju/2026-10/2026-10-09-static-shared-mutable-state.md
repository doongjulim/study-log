---
date: 2026-10-09
type: raw
author: dongju
tags: [static, 공유 가변 상태, shared mutable state, static final, SimpleDateFormat, DateTimeFormatter, 스레드 안전, thread safety, thread-unsafe, 경쟁 조건, race condition, Calendar, 날짜 포맷, date format, 불변 객체, immutable, 얕은 불변, shallow immutability, final 은 참조만 고정, 동시 요청, concurrent request, 간헐적 버그, NumberFormatException, 동일성, identity]
topic: static final SimpleDateFormat 은 왜 위험한가 — static 은 JVM 전체 스레드가 공유하고, final 은 참조만 고정할 뿐 내부 가변 상태(Calendar)는 경쟁 조건에 노출된다
summary: 같은 Date 에 같은 문자열을 내니 순수 함수처럼 보이지만, SimpleDateFormat 은 내부 Calendar 필드를 바꿔 가며 계산하는 가변 객체다. static 으로 하나만 두고 여러 요청 스레드가 동시에 format 하면 A 가 세팅한 날짜를 B 가 덮어써 다른 글의 날짜가 찍히거나 parse 에서 간헐적 예외가 나며 로컬에선 재현되지 않는다. static final 은 다른 객체를 가리키지 못하게 할 뿐 내부 변경은 못 막는다(얕은 불변). 해결은 불변 객체인 DateTimeFormatter. static 필드에는 가변 상태를 두지 않는다.
source: session
distilled: true
distilled_at: 2026-10-09
---

## 배운 개념

### 1. 순수 함수처럼 보이는 함정

```java
public final class DateUtils {
    private static final SimpleDateFormat FORMAT = new SimpleDateFormat("yyyy-MM-dd");

    public static String format(Date date) {
        return FORMAT.format(date);   // 같은 Date → 같은 문자열… 이니까 순수 함수?
    }
}
```

- `FORMAT` 은 `static` → JVM 에 **딱 하나**, 모든 요청 스레드가 공유.
- `SimpleDateFormat` 은 내부에 **`Calendar` 필드**를 들고, `format()` 할 때마다 그 필드를 **바꿔 가며** 계산 → 가변 객체.

### 2. 동시 요청 시 — 경쟁 조건

```
         스레드 A (글 1: 10/01)                 스레드 B (글 2: 12/25)
 t1   FORMAT.calendar ← 10/01 로 세팅
 t2                                         FORMAT.calendar ← 12/25 로 덮어씀
 t3   calendar 읽어서 문자열 생성 → "2026-12-25"   ❌ 글 1인데 다른 날짜!
 t4                                         "2026-12-25"
```

10/7 에 본 ArrayList 동시 add 와 같은 구조 ([[race-condition-lost-update]]):

- 화면에 **엉뚱한 날짜**가 찍힘 (에러 없이 조용히)
- `parse()` 쪽에선 `NumberFormatException` 등 **이상한 예외가 간헐적으로**
- 로컬 단독 테스트로는 **재현 안 됨**

### 3. `final` 인데 왜 바뀌나 — 얕은 불변

- `static final` 은 `FORMAT` 이 **다른 객체를 가리키지 못하게** 고정할 뿐.
- **객체 내부 필드** 변경은 못 막는다 → 9/26 의 **얕은 불변** 과 같은 이야기 ([[shallow-immutability-defensive-copy]]).

### 4. 해결

```java
private static final DateTimeFormatter FORMAT = DateTimeFormatter.ofPattern("yyyy-MM-dd");   // ✅ 불변, 스레드 안전
```

(대안: 매 호출마다 `new SimpleDateFormat` / `ThreadLocal<SimpleDateFormat>` — Java 8+ 에선 DateTimeFormatter 가 정답)

> **기준 4. static 필드에 가변 상태를 두지 않는다.** `final` 이어도 내부가 가변이면 안 된다. static 은 JVM 전체의 모든 스레드가 공유하기 때문이다 ([[static-util-vs-spring-bean]]).

## 오답·헷갈린 점

- **전**: "값이 **동일성**이 보장되지 않을 수 있음" → **후**: 방향은 맞지만 용어 오류 — 자바에서 "동일성" 은 `==`(같은 객체인가). 정확히는 공유 가변 상태(Calendar)에 대한 **경쟁 조건** → 다른 요청의 날짜로 **덮어써짐**.
- 연결 고리 (스스로 못 짚은 부분): `final` 이라 안전하다는 착각 → final 은 참조만 고정 (얕은 불변).

## Q&A

- **Q. static final SimpleDateFormat 을 여러 스레드가 쓰면?** A. 내부 Calendar 경쟁 조건 → 엉뚱한 날짜, 간헐적 예외.
- **Q. final 인데 왜?** A. final 은 참조 고정, 내부 상태 변경은 못 막음.
- **Q. 해결은?** A. 불변 `DateTimeFormatter`.

## 다음 학습

- `ThreadLocal` 의 원리와 스레드 풀에서의 누수 위험
- 불변 객체 설계 조건 (final 클래스, private final 필드, 방어적 복사)
- 스프링 싱글턴 빈에 가변 필드를 두면 안 되는 이유와의 대응 (빈도 static 과 같은 공유 문제)
