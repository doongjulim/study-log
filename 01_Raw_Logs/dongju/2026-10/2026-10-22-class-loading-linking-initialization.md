---
date: 2026-10-22
type: raw
author: dongju
tags: [클래스로딩, class-loading, 클래스로더, class-loader, 로딩, loading, 링킹, linking, 검증, verification, bytecode-verifier, 준비, preparation, 해석, resolution, 초기화, initialization, static초기화, static-initializer, static블록, clinit, 기본값, default-value, 지연로딩, lazy-loading, 클래스패스, classpath, -cp, 정규화된이름, FQCN, fully-qualified-class-name, 패키지, package, Could-not-find-or-load-main-class, ExceptionInInitializerError, NoClassDefFoundError, Could-not-initialize-class, 초기화상태, 한번만, 첫요청지연, cold-start, DispatcherServlet, load-on-startup, jar서명, 무결성, 면접답변]
topic: java 명령부터 main() 실행 직전까지 — 클래스 로딩 → 링킹(검증·준비·해석) → 초기화, 그리고 초기화가 한 번만·실패도 기록된다는 성질
summary: java com.study.Hello 를 실행하면 JVM 은 클래스패스 루트에서 패키지를 폴더 경로로 바꿔 .class 를 찾아 로딩하고, 바이트코드가 안전 규칙을 지키는지 검증한 뒤, static 변수에 타입별 기본값(0/0L/false/null)을 채우는 준비를 거쳐, 클래스를 처음 쓸 때 내 코드(static 대입·static 블록)를 실행하는 초기화를 클래스당 딱 한 번 한다. 클래스는 처음 쓰일 때 로딩되므로 배포 직후 첫 요청이 느리고, 초기화가 실패하면 그 상태가 기록돼 이후엔 원인 없는 NoClassDefFoundError 가 난다.
source: session
distilled: false
---

## 배운 개념

### 1. 전체 흐름

```
java com.study.Hello
 → java 런처(네이티브 프로그램) 가 JVM 기동
 → [로딩]   클래스패스 루트 + com/study/Hello.class 를 찾아 읽음
 → [링킹]   ① 검증(Verification): 바이트코드가 안전 규칙을 지키는가
            ② 준비(Preparation): static 변수 메모리 할당 + 타입별 기본값
            ③ 해석(Resolution): 심볼 참조 → 실제 참조   ※ 인터뷰에서 미검증, 보충
 → [초기화] static 필드 대입 · static {} 블록 실행 — 클래스당 딱 한 번
 → main() 실행 (인터프리터 → 핫스팟은 JIT)
```

- **지연 로딩(lazy loading)**: JVM 은 시작할 때 모든 클래스를 올리지 않는다. `Util`, `String`, `System` 등도 **처음 쓰일 때** 로딩·초기화된다.

### 2. 로딩 — 클래스 이름과 클래스패스

| | 형태 |
|---|---|
| 소스 파일 | `com/study/Hello.java` |
| 클래스 파일 | `com/study/Hello.class` |
| 클래스 이름 (FQCN, JVM 이 찾는 것) | `com.study.Hello` — **확장자 없음** |

```bash
# 클래스패스 루트(com/ 폴더가 들어 있는 위치)에서
java com.study.Hello          # ✅ 클래스 이름을 넘긴다
java com.study.Hello.class    # ❌ "com.study.Hello 패키지의 class 라는 클래스"를 찾으려 함
```

- JVM 은 **클래스패스 루트**에서 출발해 패키지 이름을 폴더 경로로 바꿔 내려간다.
- `Hello.class` 가 있는 폴더 **안에서** `java Hello` → 이름도 틀리고(`Hello` ≠ `com.study.Hello`) 출발점도 한 단계 깊어서 `Could not find or load main class Hello`.

### 3. 링킹 ① 검증 — "누가 만들었나"가 아니라 "안전한가"

- 검증기는 타입 일치, 스택 오버/언더플로, 점프 대상이 메서드 안인지 등 **실행해도 안전한 규칙**을 지키는지 본다.
- 손으로 작성했어도 규칙을 지키면 통과, 버그 난 컴파일러가 만든 것이라도 규칙을 어기면 거부.
- **변조 여부(무결성)** 는 검증기의 일이 아니다 → jar 서명이라는 별개 장치.

### 4. 링킹 ② 준비 vs 초기화

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
| 준비 직후 | **`0L`** (`long` 은 기본형 → null 불가) | 메모리 + 타입별 기본값(0 / 0L / false / null). 내 코드는 아직 한 줄도 안 돎 |
| 초기화 후 | 실행한 순간의 시각 | 내가 쓴 `= System.currentTimeMillis()` 를 **그제서야** 실행 |

- 초기화가 컴파일 때 일어난다면 → 매번 컴파일 시각이 찍혀야 하고, `static Connection conn = DriverManager.getConnection(...)` 이면 `javac` 순간 개발 PC 가 운영 DB 에 접속해야 한다 → 모순. **초기화는 실행 중, 클래스를 처음 쓸 때.**

### 5. 초기화는 클래스당 한 번 — JVM 이 보장한다

```java
class Util {
    static int callCount = 0;
    static { System.out.println("Util 초기화!"); }
    static void helper() { callCount++; }
}
// main: Util.helper(); Util.helper(); Util.helper();  → "Util 초기화!" 1번
```

- 이유는 "값이 이미 들어가서"가 아니다 (변수 없는 `static {}` 블록도 한 번만 돈다).
- JVM 이 **클래스마다 초기화 상태(전 / 진행 중 / 완료 / 실패)를 기록**하고 한 번만 실행되도록 **스펙으로 보장**한다. 여러 스레드가 동시에 처음 접근해도 한 스레드만 실행, 나머지는 대기.
- 매 호출마다 초기화된다면 `callCount = 0` 으로 계속 리셋 → static 공유 카운터·싱글톤 자체가 성립 불가.

### 6. 트레이드오프 · 예외 상황

**첫 요청이 느린 이유** — JIT 워밍업 + 그 요청 경로에서 처음 쓰이는 클래스들의 로딩·검증·준비·초기화 비용.
- 예: Spring Boot `DispatcherServlet` 은 기본적으로 **첫 요청 때** 초기화된다(첫 요청 로그의 `Initializing Servlet 'dispatcherServlet'`). `spring.mvc.servlet.load-on-startup=1` 로 기동 시점으로 당길 수 있다.

**초기화 실패는 기록된다**

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

- JVM 은 **재시도하지 않는다** — 상태가 "실패"로 기록됐으므로.
- 함정: `.class` 가 멀쩡히 있는데 "클래스 정의를 못 찾는다"는 에러 → 클래스패스 문제로 오해하기 쉽다.
- 대응: `NoClassDefFoundError` 를 보면 로그를 거슬러 **첫 `ExceptionInInitializerError`** 를 찾는다. 복구는 원인 수정 후 **재시작**뿐.

### 면접 답변

> "`java` 명령을 실행하면 JVM 이 클래스패스에서 패키지 경로를 따라 .class 파일을 찾아 로딩하고, 링킹 단계에서 바이트코드가 안전 규칙을 지키는지 검증한 뒤 static 변수에 기본값을 채웁니다. 그다음 클래스를 처음 사용할 때 static 필드 대입과 static 블록을 실행하는 초기화를 클래스당 한 번만 합니다. 클래스는 처음 쓰일 때 로딩되기 때문에 배포 직후 첫 요청이 느리고, 초기화 중 예외가 나면 그 실패가 기록돼서 이후 접근은 원인 정보 없는 NoClassDefFoundError 가 나기 때문에 첫 ExceptionInInitializerError 로그를 찾아야 합니다."

## 오답·헷갈린 점

- **전**: 컴파일 후 바로 "인터프리터로 실행"으로 건너뜀 → **후**: `java` 입력과 main 첫 줄 사이에 **로딩 → 링킹(검증·준비·해석) → 초기화**가 있다.
- **전**: 로딩 = "저장소에서 클래스 파일을 가져오는 과정" → **후**: 방향은 맞음. "저장소"는 **클래스패스**, 패키지 → 폴더 경로로 찾는다.
- **전**: 링킹 = "java 와 해당 클래스 파일을 연결해 주는 것" → **후**: 모호. **검증 → 준비 → 해석**.
- **전**: 검증 = "변조가 되지 않았는지 검증" → **후**: 검증기는 **안전 규칙 준수**를 본다. 변조 여부는 jar 서명.
- **전**: "초기화는 컴파일 과정에서 실행됨" → **후**: **실행 중**, 클래스를 처음 쓸 때. (현재 시각·DB 접속 반례로 스스로 모순 확인)
- **전**: 준비 직후 `long startedAt` = null → **후**: 규칙(참조 null, int 0)은 알았지만 적용 오류. `long` 은 기본형 → **0L**.
- **전**: `Util.helper()` 3번 호출 → 초기화 "3번" → **후**: **1번**. 매번이면 `callCount` 가 계속 0 으로 리셋된다.
- **전**: 한 번인 이유 = "이미 값이 들어간 변수에 다시 초기화할 필요가 없어서" → **후**: 변수 없는 static 블록도 한 번. JVM 이 **초기화 상태를 기록해 스펙으로 보장**.
- **전**: 클래스 진짜 이름 = `com.study.Hello.java`, `com.study.Hello.class` 로 실행 → **후**: 이름에는 확장자가 없다. `java com.study.Hello`.
- **전**: 초기화 실패 후 재접근 = "바로 실패를 뱉는다" → **후**: 방향은 맞음. **어떤** 에러인지가 디버깅의 핵심 — 2번째부터는 `NoClassDefFoundError: Could not initialize class`, 원인은 첫 로그에만.
- ⚠️ 반복 테마: "연결해 준다", "저장소", "바로 실패" 처럼 **동사·명사가 뭉뚱그려진 채 끝남**. 이전 세션들의 "막는다에서 멈춤"과 같은 계열 — 단계 이름 / 대상 / 결과(에러 이름)까지 붙여서 답할 것. 반면 **구체적 반례(현재 시각, DB 접속, callCount 리셋)를 주면 스스로 모순을 잡아냄** → 혼자 공부할 때도 "이게 맞다면 무슨 일이 생기나" 반례부터 세워 볼 것.

## Q&A

**Q. `java Hello` 를 입력하고 main 첫 줄이 실행되기 전까지 무슨 일이 일어나나?**
A. JVM 기동 → 클래스패스에서 .class 로딩 → 링킹(검증·준비·해석) → 초기화(static 대입·static 블록) → main 실행.

**Q. `package com.study;` 인 Hello 를 그 폴더 안에서 `java Hello` 로 실행하면 왜 못 찾나?**
A. 클래스 이름은 `com.study.Hello` 이고, JVM 은 클래스패스 루트에서 `com/study/Hello.class` 경로를 따라간다. 루트에서 `java com.study.Hello` 로 실행해야 한다.

**Q. 조작된 .class 를 JVM 이 그냥 실행하나?**
A. 아니다. 링킹의 검증 단계에서 바이트코드가 안전 규칙(타입·스택·점프)을 지키는지 확인하고 어기면 거부한다.

**Q. 준비 직후 `static long startedAt` 의 값은?**
A. `0L`. 준비는 기본값만 채우고, 내 코드의 대입은 초기화 단계에서 실행된다.

**Q. 초기화는 언제, 몇 번?**
A. 실행 중 클래스를 처음 쓸 때, 클래스당 한 번. JVM 이 초기화 상태를 기록해 보장한다.

**Q. 배포 직후 첫 요청이 느린 이유 두 가지는?**
A. JIT 워밍업 전이라 인터프리터로 실행 + 그 경로에서 처음 쓰이는 클래스들의 로딩·초기화 비용 (예: DispatcherServlet 첫 요청 초기화).

**Q. static 초기화가 실패한 클래스에 두 번째로 접근하면?**
A. 재시도 없이 `NoClassDefFoundError: Could not initialize class` — 원인은 첫 `ExceptionInInitializerError` 로그에만 있다. 재시작해야 복구.

## 다음 학습

- 클래스 로더 계층(Bootstrap → Platform → Application)과 **부모 위임 모델**
- `ClassNotFoundException` vs `NoClassDefFoundError` 구분
- 링킹의 **해석(Resolution)** 단계와 상수 풀(constant pool)
- Java 11 소스 파일 직접 실행(`java Hello.java`)이 이 과정을 어떻게 단축하는가
- distill 시: 기존 `02_Refined/jvm-execution-pipeline.md` 의 "클래스 로더: 로딩 → 링킹 → 초기화" 한 줄 → 별도 원자 노트 후보(class-loading-lifecycle), `interpreter-and-jit.md` 워밍업 항목에 클래스 로딩 비용 enrich 후보
