---
date: 2026-10-22
type: raw
author: dongju
tags: [JDK, JRE, JVM, Java Development Kit, Java Runtime Environment, 자바가상머신, WORA, Write Once Run Anywhere, 플랫폼 독립성, platform independence, 플랫폼 종속성, ISA, 명령어 세트, instruction set, 기계어, machine code, 바이트코드, bytecode, class파일, javac, 표준 클래스 라이브러리, 인터프리터, interpreter, JIT, Just-In-Time, HotSpot, 핫스팟, 워밍업, warm-up, 오토스케일링, autoscaling, AOT, Ahead-Of-Time, GraalVM, Native Image, jlink, 멀티스테이지 빌드, multi-stage build, 시스템 콜, system call, ELF, PE, JNI]
topic: JDK·JRE·JVM 포함 관계와 WORA 의 정확한 의미 — 플랫폼 종속성은 사라진 게 아니라 JVM 으로 옮겨갔다
summary: C 실행 파일은 CPU(ISA)와 OS(실행 파일 형식·시스템 콜·라이브러리) 조합에 묶이지만, 자바는 가상 CPU용 기계어인 바이트코드를 만들고 플랫폼별 JVM 이 이를 실행해 종속성을 대신 흡수한다. JVM 은 인터프리터로 시작해 핫스팟만 JIT 컴파일하므로 워밍업 비용이 있고, 이를 없애는 AOT(GraalVM Native Image)는 WORA 를 포기하는 선택이다.
source: session
distilled: true
distilled_at: 2026-10-22
---

## 배운 개념

### 1. 출발점 — C 프로그램은 왜 다른 환경에서 안 도는가

C 실행 파일은 **CPU + OS 조합**에 묶여 있다. 원인은 두 축이다.

- **CPU 축**: C 컴파일러는 CPU 가 직접 실행하는 **기계어**를 만든다. 기계어는 CPU 의 명령어 세트(**ISA**)마다 다르다. x86 기계어는 ARM CPU 가 읽지 못한다 → OS 가 같은 Windows 11 이어도 x86 `.exe` 는 ARM 노트북에서 네이티브로 안 돈다.
- **OS 축**
  - 실행 파일 형식: Windows **PE**(`.exe`) / Linux **ELF** / macOS **Mach-O**
  - **시스템 콜**: 파일 열기·출력 등 OS 에 요청하는 인터페이스가 OS 마다 다름
  - 런타임 라이브러리: `.dll` / `.so` 등 의존 라이브러리가 다름

### 2. JDK ⊃ JRE ⊃ JVM

| 구성 | 내용 | 필요한 곳 |
|---|---|---|
| **JVM** | 바이트코드를 실행하는 엔진 | — |
| **JRE** | JVM + **표준 클래스 라이브러리**(`String`, `ArrayList`, `System` 등 `java.lang`/`java.util` 구현체) | 운영 서버 (실행만) |
| **JDK** | JRE + **개발 도구**(`javac`, `jar`, `javadoc`, `jdb`) | 개발 PC (컴파일·테스트) |

- 운영 서버에서는 컴파일할 일이 없으니 `javac` 가 필요 없다 → JRE 안에 JDK 가 있을 수 없다.
- 실무 보충: Java 11 부터 오라클은 별도 JRE 배포를 중단했다. `jlink` 로 앱에 필요한 모듈만 담은 커스텀 런타임을 만들 수 있다. Temurin 등은 `eclipse-temurin:21-jre` 이미지를 여전히 제공하므로 **빌드는 JDK, 실행은 JRE** 멀티스테이지 빌드가 흔하다.

### 3. WORA 의 정확한 의미

```
.java ──(javac)──▶ .class (바이트코드: x86용도 ARM용도 아닌, 가상 CPU = JVM 용 기계어)
                      │
        ┌─────────────┼─────────────┐
   JVM(Win/x64)  JVM(Linux/aarch64)  JVM(macOS/aarch64)   ← JVM 은 C/C++ 네이티브 프로그램이라 OS+CPU 별로 따로 존재
```

> **플랫폼 종속성이 사라진 게 아니라, "내 프로그램"에서 "JVM"으로 옮겨간 것이다.**
> 개발자는 `.class` 하나만 만들고, 플랫폼 포팅은 JVM 을 만드는 쪽이 대신 한다.

- JDK 다운로드 시 고르는 것도 **OS × CPU 아키텍처**(x64 / aarch64) 조합.
- JVM 이 바이트코드를 번역하는 대상은 **CPU 기계어**이고, OS 관련 처리는 JVM 이 시스템 콜을 대신 호출해서 흡수한다.

### 4. 실행 메커니즘 — 인터프리터 + JIT (HotSpot)

```
.class → JVM 인터프리터로 해석 실행
       → 메서드 호출 횟수·반복문 회전 수 카운팅 → 임계치 넘는 핫스팟 감지
       → JIT 가 핫스팟만 기계어로 컴파일 → 코드 캐시에 저장
       → 이후 번역 없이 네이티브 속도로 실행
```

- **인터프리터**: 시작이 빠르다. 한 줄 번역 → 실행 → 결과를 버리기 때문에 반복 코드는 매번 재번역 → **CPU 시간 낭비, 느림**.
- **JIT**: 실행이 빠르다. 대신 **컴파일 비용(CPU) + 코드 캐시 메모리**가 든다.
- 그래서 **전부가 아니라 핫스팟만** 컴파일한다. 한 번만 실행되는 초기화 코드 같은 걸 컴파일하면 컴파일 시간과 메모리만 손해다. "HotSpot JVM" 이라는 이름이 여기서 왔다.

### 5. 한계·트레이드오프

- **워밍업**: 배포 직후 JVM 은 아직 핫스팟을 모르니 대부분 인터프리터로 돌고, 백그라운드 JIT 컴파일로 CPU 도 쓴다 → 느리다. 몇 시간 뒤엔 컴파일 결과가 쌓여 빨라진다.
  - **오토스케일링 함정**: 트래픽 폭주로 새로 뜬 서버가 하필 가장 바쁜 순간에 가장 느린 상태다.
- **AOT (GraalVM Native Image)**: 실행 전에 바이트코드를 특정 CPU+OS 용 실행 파일로 통째로 컴파일 → 시작 빠르고 워밍업 거의 없음. 대가는 **WORA 포기** — C 프로그램처럼 다시 CPU+OS 에 묶인다.
- **WORA 가 깨지는 코드** — JVM 이 보장하는 범위 밖으로 나가는 순간 종속성이 내 프로그램으로 돌아온다.

| 깨지는 코드 | 이유 | 해결 |
|---|---|---|
| `"C:\\data\\upload\\a.txt"` 하드코딩 | 경로 구분자·루트 구조 차이 | `Path.of("data", "upload", "a.txt")`, 경로는 설정으로 분리 |
| `Runtime.exec("cmd /c dir")` | `cmd` 는 Windows 전용 | OS 명령어 호출 지양 / OS 분기 |
| `Logo.PNG` 저장 후 `logo.png` 로 읽기 | Windows·macOS 는 대소문자 무시, Linux 는 구분 | 파일명 규칙 통일 |
| JNI 로 `.dll` 호출 | 네이티브 라이브러리는 CPU+OS 종속 | 플랫폼별 `.so`/`.dylib` 준비 |

## 오답·헷갈린 점

- **C 컴파일러가 바이트코드를 만든다** → C 는 **CPU 기계어**, 바이트코드는 `javac` 의 결과물.
- **x86 `.exe` 도 OS 만 같으면 ARM 에서 네이티브로 돈다** → 기계어는 ISA 마다 다르므로 안 돈다.
- **플랫폼 종속성 = OS 때문** → **CPU(ISA) + OS** 두 축. *(인터뷰 내내 반복된 습관 — JVM 도 "운영체제별로 다르다"고만 말함)*
- **JRE = JVM + JDK, JDK = JAVA + JAVAC** → 포함 관계가 뒤집힘. **JDK ⊃ JRE ⊃ JVM**. JRE = JVM + 표준 클래스 라이브러리, JDK = JRE + 개발 도구.
- **JRE 에 필요한 건 JVM + 운영체제** → OS 는 원래 깔려 있는 것이지 JRE 구성요소가 아님. 빠진 건 표준 클래스 라이브러리.
- **JVM 이 자바 프로그램을 컴파일한다** → 컴파일(소스→바이트코드)은 `javac`, JVM 은 **실행**(인터프리터 + JIT).
- **바이트코드는 `javac` 로 저장된다** → `javac` 는 컴파일러 이름, 결과물 확장자는 **`.class`**.
- **JVM 이 운영체제에 맞게 번역한다** → 번역 대상은 **CPU 기계어**, OS 쪽은 시스템 콜로 흡수.
- **인터프리터의 문제는 용량 / "풀"이 늘어남** → 번역 결과를 버리므로 메모리는 안 쌓인다. 낭비되는 건 **CPU 시간(속도)**.
- **"인터프리터로 컴파일", "배포 직후 계속 컴파일"** → 인터프리터는 컴파일하지 않고 **해석(interpret)** 한다. 해석 ≠ 컴파일. *(반복된 습관)*
- **JDK 는 Java Developer Kit** → **Development** Kit.

## Q&A

**Q. C 실행 파일이 다른 환경에서 안 도는 이유는?**
A. CPU 축(기계어가 ISA 마다 다름)과 OS 축(실행 파일 형식 PE/ELF/Mach-O, 시스템 콜, 런타임 라이브러리) 두 가지. 결국 CPU+OS 조합에 묶인다.

**Q. 개발 PC 와 운영 서버에는 각각 무엇이 필요한가?**
A. 개발 PC 는 컴파일·테스트를 해야 하니 JDK. 운영 서버는 실행만 하므로 JVM + 표준 클래스 라이브러리, 즉 JRE 면 충분하다.

**Q. 바이트코드는 x86용인가 ARM용인가?**
A. 둘 다 아니다. JVM 이라는 가상 CPU 용 기계어다. 그래서 어느 플랫폼이든 그 플랫폼용 JVM 만 있으면 같은 `.class` 가 돈다.

**Q. Windows x86 JVM 과 Linux ARM JVM 은 같은 프로그램인가?**
A. 다르다. JVM 자체가 네이티브 프로그램이라 OS × CPU 조합마다 따로 있다. WORA 는 종속성을 JVM 쪽으로 옮긴 것.

**Q. 왜 처음부터 전부 JIT 컴파일하지 않는가?**
A. 한 번만 실행되는 코드가 많아서, 다 컴파일하면 컴파일 시간과 코드 캐시 메모리만 손해다. 호출 횟수를 세서 핫스팟만 컴파일한다.

**Q. 배포 직후와 몇 시간 뒤의 응답 속도가 같은가? 오토스케일링에선?**
A. 다르다. 직후엔 인터프리터 위주 + 백그라운드 JIT 로 느리고, 시간이 지나면 빨라진다. 오토스케일링으로 새로 뜬 서버는 트래픽이 가장 몰린 순간에 가장 느린 상태라 문제가 된다.

**Q. GraalVM Native Image 를 쓰면 무엇을 포기하나?**
A. WORA. 특정 CPU+OS 용 실행 파일이 되므로 C 프로그램처럼 다시 플랫폼에 묶인다.

**Q. 자바인데 Windows 에선 되고 Linux 에선 깨지는 코드는?**
A. 경로 하드코딩, `Runtime.exec` 로 OS 명령 호출, 파일명 대소문자 불일치, JNI 네이티브 라이브러리 — JVM 보장 범위 밖으로 나가는 코드.

## 다음 학습

- 워밍업 완화책: JIT 단계(C1/C2, Tiered Compilation), CDS·AppCDS, CRaC, 배포 시 워밍업 트래픽 / readiness probe 지연
- GraalVM Native Image 로 Spring Boot 빌드 시 리플렉션·프록시 제약 (Spring AOT)
- `jlink` + 멀티스테이지 Docker 빌드로 런타임 이미지 크기 줄이기
- distill 시: 기존 `02_Refined/jvm-execution-pipeline.md` 요약의 "OS 별로 따로 구현된 JVM" → **OS × CPU** 로 정정 후보, `interpreter-and-jit.md` 에 워밍업·오토스케일링·AOT 트레이드오프 enrich 후보
