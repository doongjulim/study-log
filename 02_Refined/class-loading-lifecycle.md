---
type: refined
slug: class-loading-lifecycle
tags: [클래스로딩, class-loading, 클래스로더, class-loader, 클래스생명주기, class-lifecycle, 로딩, loading, 링킹, linking, 검증, verification, bytecode-verifier, 바이트코드검증, 준비, preparation, 해석, resolution, 초기화, initialization, 클래스초기화, class-initialization, static초기화, static-initializer, static블록, clinit, 기본값, default-value, 지연로딩, lazy-loading, 클래스패스, classpath, -cp, 정규화된이름, FQCN, fully-qualified-class-name, 패키지경로, Could-not-find-or-load-main-class, ExceptionInInitializerError, NoClassDefFoundError, Could-not-initialize-class, 초기화상태, 한번만실행, 첫요청지연, cold-start, DispatcherServlet, load-on-startup, jar서명, 면접답변]
topic: "java 명령부터 main() 직전까지 — 로딩 → 링킹(검증·준비·해석) → 초기화, 그리고 초기화는 클래스당 한 번이고 실패도 기록된다"
summary: "JVM 은 클래스패스 루트에서 패키지를 폴더 경로로 바꿔 .class 를 찾아 로딩하고, 바이트코드가 안전 규칙을 지키는지 검증한 뒤, static 변수에 타입별 기본값을 채우는 준비를 거쳐, 클래스를 처음 쓸 때 static 대입과 static 블록을 실행하는 초기화를 클래스당 딱 한 번 한다. 클래스는 처음 쓰일 때 로딩되므로 배포 직후 첫 요청이 느리고, 초기화가 실패하면 그 상태가 기록돼 이후 접근은 원인 정보 없는 NoClassDefFoundError 가 난다."
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/javase/specs/jvms/se17/html/jvms-5.html
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-12.html
updated: 2026-10-22
---

# 클래스 로딩 생명주기 — 로딩 · 링킹 · 초기화

## 개념 정의

```
java com.study.Hello
 → java 런처(네이티브 프로그램)가 JVM 기동
 → [로딩]   클래스패스 루트 + com/study/Hello.class 를 찾아 읽음
 → [링킹]   ① 검증(Verification): 바이트코드가 안전 규칙을 지키는가
            ② 준비(Preparation): static 변수 메모리 할당 + 타입별 기본값
            ③ 해석(Resolution): 심볼 참조 → 실제 참조
 → [초기화] static 필드 대입 · static {} 블록 실행 — 클래스당 딱 한 번
 → main() 실행 (인터프리터 → 핫스팟은 JIT: [[interpreter-and-jit]])
```

- **지연 로딩(lazy loading)**: JVM 은 시작할 때 모든 클래스를 올리지 않는다. 사용자 클래스도 `String`·`System` 같은 표준 클래스도 **처음 쓰일 때** 로딩되고 초기화된다.

## 작동 원리

### 로딩 — 클래스 이름과 클래스패스

| | 형태 |
|---|---|
| 소스 파일 | `com/study/Hello.java` |
| 클래스 파일 | `com/study/Hello.class` |
| 클래스 이름 (FQCN, JVM 이 찾는 것) | `com.study.Hello`, **확장자 없음** |

```bash
# 클래스패스 루트(com/ 폴더가 들어 있는 위치)에서
java com.study.Hello          # ✅ 클래스 이름을 넘긴다
java com.study.Hello.class    # ❌ "com.study.Hello 패키지의 class 라는 클래스"를 찾으려 함
```

JVM 은 **클래스패스 루트**에서 출발해 패키지 이름을 폴더 경로로 바꿔 내려간다. `Hello.class` 가 있는 폴더 **안에서** `java Hello` 를 실행하면 이름도 틀리고(`Hello` ≠ `com.study.Hello`) 출발점도 한 단계 깊어서 `Could not find or load main class Hello` 가 난다.

### 링킹 ① 검증 — "누가 만들었나"가 아니라 "안전한가"

- 검증기는 타입이 맞는지, 스택이 넘치지 않는지, 점프 대상이 메서드 안에 있는지 같은 **실행해도 안전한 규칙**을 확인한다.
- 손으로 쓴 바이트코드라도 규칙을 지키면 통과하고, 버그 난 컴파일러가 만든 것이라도 규칙을 어기면 거부한다.
- **변조 여부(무결성)는 검증기의 일이 아니다.** 그건 jar 서명이 맡는다.

### 링킹 ② 준비 vs 초기화

```java
public class Hello {
    static long startedAt = System.currentTimeMillis();
    public static void main(String[] args) {
        System.out.println(startedAt);
    }
}
```

| 단계 | `startedAt` 의 값 | 하는 일 |
|---|---|---|
| 준비 직후 | **`0L`** (`long` 은 기본형이라 null 불가) | 메모리 할당 + 타입별 기본값(0 / 0L / false / null). 내 코드는 아직 한 줄도 실행되지 않음 |
| 초기화 후 | 실행한 순간의 시각 | 내가 쓴 `= System.currentTimeMillis()` 를 **그제서야** 실행 |

초기화가 컴파일할 때 일어난다면 매번 컴파일한 시각이 찍혀야 하고, `static Connection conn = DriverManager.getConnection(...)` 이면 `javac` 를 돌리는 순간 개발 PC 가 운영 DB 에 접속해야 한다. 그래서 **초기화는 실행 중, 클래스를 처음 쓸 때** 일어난다.

### 초기화는 클래스당 한 번 — JVM 이 보장한다

```java
class Util {
    static int callCount = 0;
    static { System.out.println("Util 초기화!"); }
    static void helper() { callCount++; }
}
// Util.helper() 를 세 번 호출해도 "Util 초기화!" 는 1번
```

- 이유는 "값이 이미 들어가서"가 아니다. 변수가 없는 `static {}` 블록도 한 번만 돈다.
- JVM 이 **클래스마다 초기화 상태**(초기화 전 / 진행 중 / 완료 / 실패)를 기록하고 **스펙으로 한 번만 보장**한다. 여러 스레드가 동시에 처음 접근해도 한 스레드만 실행하고 나머지는 기다린다.
- 호출할 때마다 초기화된다면 `callCount` 가 계속 0 으로 리셋되어 static 공유 상태나 싱글톤 자체가 성립하지 않는다 ([[singleton-bean-stateful]]).

## 트레이드오프 / 한계

### 지연 로딩의 대가 — 첫 요청이 느리다

배포 직후 첫 요청에는 JIT 워밍업 전 인터프리터 실행에 더해, **그 요청 경로에서 처음 쓰이는 클래스들의 로딩·검증·준비·초기화 비용**이 붙는다.
예를 들어 Spring Boot 의 `DispatcherServlet` 은 기본 설정에서 **첫 요청 때** 초기화된다(첫 요청 로그의 `Initializing Servlet 'dispatcherServlet'`). `spring.mvc.servlet.load-on-startup=1` 로 기동 시점으로 당길 수 있다.

### 초기화 실패는 기록된다 — `NoClassDefFoundError` 함정

```java
class Config {
    static final String DB_URL = loadFromFile("db.properties"); // 파일 없으면 예외
}
```

```
1번째 접근: java.lang.ExceptionInInitializerError
              Caused by: java.io.FileNotFoundException: db.properties   ← 진짜 원인
2번째 접근: java.lang.NoClassDefFoundError: Could not initialize class Config
              (원인 정보 없음)
```

- 상태가 "실패"로 기록됐으므로 JVM 은 **재시도하지 않는다.**
- `.class` 파일이 멀쩡히 있는데 "클래스 정의를 못 찾는다"는 에러가 나서 **클래스패스 문제로 오해하기 쉽다.**
- `NoClassDefFoundError` 가 보이면 로그를 거슬러 **첫 `ExceptionInInitializerError`** 를 찾는다. 원인을 고친 뒤 **재시작**해야 복구된다.

> [!WARNING]
> **오답 코너**
> - **"컴파일하면 바로 인터프리터가 실행한다"** — `java` 입력과 main 첫 줄 사이에 **로딩 → 링킹(검증·준비·해석) → 초기화**가 있다.
> - **"링킹 = java 와 클래스 파일을 연결해 주는 것"** — 모호하다. **검증 → 준비 → 해석** 세 단계로 말할 것.
> - **"검증 = 변조되지 않았는지 확인"** — 검증기는 **안전 규칙 준수**를 본다. 변조 여부는 jar 서명이 맡는다.
> - **"초기화는 컴파일 과정에서 실행된다"** — **실행 중**, 클래스를 처음 쓸 때다.
> - **"준비 직후 `static long` 은 null"** — `long` 은 기본형이라 **0L** 이다.
> - **"static 초기화는 호출할 때마다 실행된다"** — 클래스당 **한 번**. JVM 이 초기화 상태를 기록해 보장한다.
> - **"클래스 이름은 `com.study.Hello.class`"** — 이름에는 확장자가 없다. `java com.study.Hello`.
> - **"초기화 실패 후 재접근도 같은 예외"** — 두 번째부터는 `NoClassDefFoundError: Could not initialize class` 이고, 원인은 첫 로그에만 있다.

## 면접 답변

> "`java` 명령을 실행하면 JVM 이 클래스패스에서 패키지 경로를 따라 .class 파일을 찾아 로딩하고, 링킹 단계에서 바이트코드가 안전 규칙을 지키는지 검증한 뒤 static 변수에 기본값을 채웁니다. 그다음 클래스를 처음 사용할 때 static 필드 대입과 static 블록을 실행하는 초기화를 클래스당 한 번만 합니다. 클래스는 처음 쓰일 때 로딩되기 때문에 배포 직후 첫 요청이 느리고, 초기화 중 예외가 나면 그 실패가 기록돼서 이후 접근은 원인 정보가 없는 NoClassDefFoundError 가 나므로 첫 ExceptionInInitializerError 로그를 찾아야 합니다."

## 복습 체크

- [ ] `java Hello` 부터 main 첫 줄까지의 단계를 하위 단계까지 순서대로 말할 수 있는가?
- [ ] `package com.study;` 인 클래스를 그 폴더 안에서 `java Hello` 로 실행하면 왜 못 찾는가?
- [ ] 바이트코드 검증기가 확인하는 것과 확인하지 않는 것은?
- [ ] 준비 직후와 초기화 후 `static long startedAt` 의 값은 각각 무엇인가?
- [ ] static 초기화가 한 번만 실행되는 진짜 이유는?
- [ ] 배포 직후 첫 요청이 느린 이유 두 가지는?
- [ ] static 초기화가 실패한 클래스에 두 번째로 접근하면 어떤 에러가 나고, 원인은 어디서 찾는가?

## 관련

[[jvm-execution-pipeline]] · [[interpreter-and-jit]] · [[jdk-jre-jvm]] · [[primitive-vs-reference-type]] · [[singleton-bean-stateful]]
