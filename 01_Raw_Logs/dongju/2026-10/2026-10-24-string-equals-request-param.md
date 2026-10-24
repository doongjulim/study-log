---
date: 2026-10-24
type: raw
author: dongju
tags: [문자열비교, string-comparison, equals, ==, 동일성, 동등성, 요청파라미터, request-param, RequestParam, 톰캣, tomcat, 바이트에서문자열, 실행중생성, 힙의새객체, 리터럴공유, 테스트우연통과, coincidental-pass, 래퍼캐시비유, NullPointerException, NPE, 리터럴앞에, literal-first-equals, Objects.equals, null안전, null-safe, 권한검사, role, 면접답변]
topic: 요청 파라미터처럼 실행 중에 만들어진 문자열은 힙의 새 객체라 리터럴과 == 로 비교하면 false — 테스트에선 리터럴끼리라 우연히 통과하므로 문자열 비교는 무조건 equals, null 안전은 Objects.equals
summary: 테스트 코드의 "ADMIN" 과 컨트롤러의 "ADMIN" 은 둘 다 리터럴이라 같은 풀 객체를 공유해 role == "ADMIN" 이 우연히 true 가 된다. 하지만 실제 요청에서 톰캣은 받은 바이트로 실행 중에 String 을 새로 만들므로 role 은 힙의 새 객체이고 == 는 false 가 되어 권한 검사가 실패한다. 문자열의 출처에 따라 == 결과가 달라지고 출처를 매번 따질 수 없으므로 내용 비교는 무조건 equals 다. role 이 null 이면 role.equals("ADMIN") 은 NPE, "ADMIN".equals(role) 은 false 이고, 양쪽 다 null 일 수 있으면 Objects.equals 를 쓴다.
source: session
distilled: true
distilled_at: 2026-10-24
---

## 배운 개념

### 1. 로컬 테스트는 되는데 실제 요청은 안 되는 코드

```java
// 컨트롤러
@GetMapping("/admin")
public String admin(@RequestParam String role) {
    if (role == "ADMIN") {               // ← 리터럴 ㉯
        return "관리자 화면";
    }
    return "권한 없음";
}

// 테스트 코드
controller.admin("ADMIN");               // ← 리터럴 ㉮
```

| | 비교하는 두 문자열 | 위치 | `role == "ADMIN"` | 화면 |
|---|---|---|---|---|
| **테스트** | 테스트 리터럴 ㉮ vs 컨트롤러 리터럴 ㉯ | 둘 다 **풀** → 같은 객체 | **true** | 관리자 화면 |
| **실제 요청** | 톰캣이 바이트로 만든 `role` vs 리터럴 ㉯ | `role` 은 **힙**(실행 중 생성), ㉯는 **풀** | **false** | 권한 없음 |

- 리터럴은 어느 클래스에 적혀 있든 JVM 의 **하나의 상수 풀**에서 같은 객체를 공유한다.
- 브라우저는 `A D M I N` **바이트**를 보낼 뿐이고, 톰캣이 요청이 들어온 순간 그 바이트로 String 을 새로 만든다 → 컴파일러가 미리 알 수 없는 값 → 힙의 새 객체 ([[2026-10-24-string-literal-pool-folding-intern]] 의 `"java" + part` 와 같은 상황).
- 테스트가 **우연히 통과**한다 — `==` 가 맞게 동작한 게 아니라 리터럴끼리라 우연히 같은 객체였을 뿐. `Long` 이 127 까지 `==` 로 통과하다 128 부터 깨지는 것과 같은 구조 ([[wrapper-cache-autoboxing]]).

> **문자열이 어디서 왔는지(리터럴 / 요청·DB·연결 결과)에 따라 `==` 결과가 달라진다. 출처를 매번 따질 수 없으니 내용 비교는 무조건 `equals`.**

### 2. `equals` 의 순서와 null

```java
role.equals("ADMIN")      // role 이 null 이면 NullPointerException
"ADMIN".equals(role)      // role 이 null 이면 false — 리터럴은 절대 null 이 아님
Objects.equals(a, b)      // 둘 다 null → true, 한쪽만 null → false
```

- 연결: [[reference-vs-value-equality]]

### 면접 답변

> "== 는 참조를 비교하기 때문에 같은 내용이라도 다른 객체면 false 입니다. 리터럴끼리는 상수 풀에서 같은 객체를 공유해서 테스트에선 우연히 true 가 나오지만, 요청 파라미터나 DB 에서 읽은 문자열은 실행 중에 새로 만든 객체라 false 가 됩니다. 그래서 문자열 비교는 항상 equals 를 쓰고, null 일 수 있으면 리터럴을 앞에 두거나 Objects.equals 를 씁니다."

## 오답·헷갈린 점

- 질문이 추상적이라 "더 구체적으로" 요청 → (a)~(d) 보기 고르기로 쪼개 진행.
- **전**: (a) 테스트의 `"ADMIN"` 과 컨트롤러의 `"ADMIN"` 은 "**하나는 풀, 하나는 힙**" → **후**: **둘 다 리터럴 → 둘 다 풀, 같은 객체.** 힙에 있는 건 **실제 요청에서 톰캣이 만든** `role`. 테스트 상황과 실제 요청 상황을 섞음.
- (b)~(d) 는 답하지 않아 정리받음: 테스트 true / 실제 요청의 role 은 힙 / 실제 요청 false → "권한 없음".
- 👍 `role.equals("ADMIN")` → NPE, `"ADMIN".equals(role)` → false — 정답. (`Objects.equals` 이름은 보충)

## Q&A

**Q. 왜 문자열 비교는 equals 로 해야 하나?**
A. == 는 참조 비교라 같은 내용이어도 다른 객체면 false. 요청·DB·런타임 연결 결과는 새 객체라 리터럴과 == 가 false 가 된다.

**Q. 그런데 테스트에서는 == 가 통과했다. 왜?**
A. 테스트의 리터럴과 컨트롤러의 리터럴이 같은 풀 객체라 우연히 true.

**Q. null 일 수 있는 문자열은 어떻게 비교하나?**
A. `"ADMIN".equals(role)` 처럼 리터럴을 앞에, 양쪽 다 null 가능하면 `Objects.equals(a, b)`.

## 다음 학습

- 문자열 switch 는 내부적으로 `hashCode()` + `equals()` 로 동작 → == 함정 없음, 단 null 이면 NPE
- enum 으로 역할을 받아 문자열 비교 자체를 없애기(`Role.valueOf`) ([[enum-constant-specific-polymorphism]])
