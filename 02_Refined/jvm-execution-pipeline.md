---
type: refined
slug: jvm-execution-pipeline
tags: [JVM, 자바가상머신, 실행파이프라인, javac, 컴파일, 바이트코드, bytecode, class파일, WORA, Write-Once-Run-Anywhere, 플랫폼독립성, 클래스로더, class-loader, 런타임데이터영역, runtime-data-area, 실행엔진, execution-engine, JRE, JDK, 플랫폼종속성, platform-dependency, ISA, 명령어세트, instruction-set, 기계어, machine-code, CPU아키텍처, x64, aarch64, 시스템콜, system-call, PE, ELF, Mach-O, JNI, 경로하드코딩, 대소문자, case-sensitivity, Runtime.exec, AOT, GraalVM, native-image]
topic: .java 가 실행되기까지의 경로와 JVM 3대 구성요소, WORA 가 성립하는 구조
summary: C 실행 파일은 CPU(명령어 세트)와 OS(실행 파일 형식·시스템 콜·라이브러리) 조합에 묶인다. 자바는 javac 가 소스를 가상 CPU(JVM)용 기계어인 바이트코드(.class)로 컴파일하고, OS × CPU 아키텍처 조합마다 따로 빌드된 JVM 이 이를 읽어 실행한다. 플랫폼 종속성이 사라진 게 아니라 내 프로그램에서 JVM 으로 옮겨간 것이 WORA 의 실체이며, JVM 이 보장하는 범위 밖(OS 명령·파일 시스템 규칙·네이티브 코드)으로 나가면 종속성이 다시 돌아온다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/javase/specs/jvms/se17/html/index.html
  - https://adoptium.net/temurin/releases/
updated: 2026-10-22
---

# JVM 실행 파이프라인과 WORA

## 개념 정의

```
Hello.java  --[javac]-->  Hello.class  --[JVM]-->  실행
(소스코드)   (컴파일러)     (바이트코드)   (가상머신)
```

**바이트코드는 실제 CPU 의 기계어도 소스코드도 아닌 JVM 전용 중간 코드**다. x86 용도 ARM 용도 아니다 — JVM 이라는 **가상 CPU 의 기계어**라고 보면 된다. OS 도 CPU 도 아니라 JVM 이 읽는다.

## 작동 원리

### 출발점 — C 실행 파일이 묶이는 두 축

C/C++ 는 소스를 **타깃 기계어로 직접** 컴파일한다. 그 결과물은 두 축에 동시에 묶인다.

| 축 | 무엇이 다른가 | 예 |
|---|---|---|
| **CPU** | 명령어 세트(ISA)가 달라 기계어가 다르다 | x86 `.exe` 는 OS 가 같은 Windows 11 이어도 ARM 노트북에서 네이티브로 안 돈다 |
| **OS** | 실행 파일 형식 | Windows **PE**(`.exe`) / Linux **ELF** / macOS **Mach-O** |
| | 시스템 콜 (파일 열기·출력 등을 OS 에 요청하는 인터페이스) | Windows API vs Linux syscall |
| | 런타임 라이브러리 | `.dll` / `.so` / `.dylib` |

그래서 OS 나 CPU 중 **하나만** 바뀌어도 다시 빌드해야 한다. 이것이 자바가 해결하려 한 Pain Point다.

### WORA 가 성립하는 구조

자바는 층을 하나 끼워 넣는다.

```
어느 플랫폼에서든 동일한 Hello.class (바이트코드)
        ↓            ↓             ↓
  JVM(Windows/x64) JVM(macOS/aarch64) JVM(Linux/aarch64)   ← 플랫폼 차이를 여기서 흡수
        ↓            ↓             ↓
  각 CPU 의 기계어로 변환되어 실행 (OS 기능은 JVM 이 시스템 콜로 대신 호출)
```

**바이트코드는 어디서나 같고, 다른 것은 JVM 쪽이다.**
JVM 자체는 C/C++ 로 만든 **네이티브 프로그램**이라 C 프로그램과 똑같이 CPU + OS 에 묶인다.
JDK 를 내려받을 때 **OS(Windows/Linux/macOS) × 아키텍처(x64/aarch64)** 조합을 고르는 이유다.

"Write Once, Run Anywhere" 는 *"자바가 플랫폼을 안 탄다"* 가 아니라
*"플랫폼을 타는 부분을 JVM 이 대신 떠맡는다"* 는 뜻이다.

> **플랫폼 종속성이 사라진 게 아니라, "내 프로그램"에서 "JVM"으로 옮겨간 것이다.**
> 개발자는 `.class` 하나만 만들고, 플랫폼별 포팅은 JVM 을 만드는 쪽이 대신 한다.

> 정정 (2026-10-22): 이전 판은 "OS 별로 따로 구현된 JVM" 이라고 적었다. JVM 은 **OS × CPU 아키텍처** 조합별로 존재한다 — 같은 OS 라도 ISA 가 다르면 기계어가 달라 다른 JVM 빌드가 필요하다(Temurin 배포 페이지의 OS·Architecture 선택 참고).

### JVM 3대 구성요소

| 구성요소 | 역할 |
|---|---|
| **클래스 로더** | `.class` 파일을 읽어 메모리에 적재 (로딩 → 링킹 → 초기화) |
| **런타임 데이터 영역** | 적재된 코드·데이터가 놓이는 메모리 공간 (스택, 힙, 메서드 영역 등) |
| **실행 엔진** | 바이트코드를 기계어로 변환해 실행 (인터프리터 + JIT) |

각각의 심화는 [[jvm-stack-and-heap]], [[interpreter-and-jit]] 참고.

## 트레이드오프 / 한계

- **실행 전 변환 층이 하나 더 있다** → 네이티브 컴파일 언어보다 시작이 느리고 메모리를 더 쓴다.
  ([[interpreter-and-jit]] 의 하이브리드 전략이 이를 완화한다)
- **JVM 을 설치해야 실행된다** — 배포물이 자기완결적이지 않다.
  (컨테이너·`jlink`·GraalVM 네이티브 이미지가 이 지점을 공략한다)
- **"어디서나 똑같이 동작"은 이상에 가깝다** — **JVM 이 보장하는 범위 밖으로 나가는 순간** 종속성이 내 프로그램으로 돌아온다.
  ("Write Once, Debug Everywhere" 라는 농담의 출처)

  | 깨지는 코드 | 이유 | 해결 |
  |---|---|---|
  | `"C:\\data\\upload\\a.txt"` 하드코딩 | 경로 구분자(`\` vs `/`)·루트 구조 차이 | `Path.of("data", "upload", "a.txt")`, 경로는 설정으로 분리 ([[externalized-configuration]]) |
  | `Runtime.exec("cmd /c dir")` | `cmd` 는 Windows 전용 OS 명령 | OS 명령 호출 지양 / OS 분기 |
  | `Logo.PNG` 로 저장, `logo.png` 로 읽기 | Windows·macOS 기본 파일 시스템은 대소문자 무시, Linux 는 구분 | 파일명 규칙 통일 — 로컬 OK·Linux 배포 후 장애의 단골 |
  | JNI 로 `.dll` 호출 | 네이티브 라이브러리는 다시 CPU + OS 종속 | 플랫폼별 `.so`/`.dylib` 준비 |

  그 밖에 기본 문자 인코딩, 스레드 스케줄링, JVM 벤더·버전 차이에서도 새어 나온다.
- **AOT(GraalVM Native Image)는 WORA 를 맞바꾸는 선택이다** — 시작이 빠르고 워밍업이 없는 대신, 결과물이 특정 CPU + OS 용 실행 파일이라 C 프로그램처럼 다시 플랫폼에 묶인다 ([[interpreter-and-jit]]).
- 바이트코드는 사람이 읽기 쉬운 구조라 **역컴파일에 취약**하다(난독화가 별도 과제가 되는 이유).

> [!WARNING]
> **오답 코너**
> - **"javac 파일로 컴파일된다"** — `javac` 는 **컴파일러(명령어) 자체**이고,
>   산출물은 **`.class` 파일(바이트코드)** 이다. 도구 이름과 결과물을 혼동하기 쉽다.
> - **"바이트코드는 기계어다"** — 아니다. CPU 가 아니라 **JVM 이 읽는 중간 코드**다.
> - **"자바는 인터프리터 언어다"** — 컴파일(javac) + 인터프리트 + JIT 컴파일이 모두 섞인 하이브리드다.
> - **"WORA 는 자바 언어의 성질"** — **JVM 이 OS × CPU 조합별로 따로 구현되어 있기 때문에** 성립하는 것이다.
> - **"C 컴파일러도 바이트코드를 만든다"** — C 는 **CPU 기계어**를, `javac` 가 **바이트코드**를 만든다.
> - **"플랫폼 종속성 = OS 차이"** — **CPU(ISA) + OS** 두 축이다. OS 가 같아도 x86 실행 파일은 ARM 에서 네이티브로 안 돈다.
> - **"JVM 이 운영체제에 맞게 번역한다"** — 번역 대상은 **CPU 기계어**다. OS 쪽은 JVM 이 시스템 콜을 대신 호출해 흡수한다.
> - **"JVM 이 자바 프로그램을 컴파일한다"** — 소스 → 바이트코드 컴파일은 **javac**. JVM 은 바이트코드를 **실행**한다(인터프리터 + JIT).
> - **"자바니까 어디서나 그대로 돈다"** — 경로 하드코딩·OS 명령·파일명 대소문자·JNI 에서 깨진다.

## 복습 체크

- [ ] `.java` 부터 실행까지의 단계를 도구 이름과 산출물까지 정확히 말할 수 있는가?
- [ ] 바이트코드가 무엇이고 누가 읽는지 설명할 수 있는가?
- [ ] C 실행 파일이 다른 환경에서 안 도는 이유를 CPU 축과 OS 축으로 나눠 말할 수 있는가?
- [ ] WORA 가 성립하는 이유를 "JVM 이 OS × CPU 조합별로 따로 있다"로 설명할 수 있는가?
- [ ] "종속성이 사라진 게 아니라 옮겨갔다"는 말을 JDK 다운로드 화면을 예로 설명할 수 있는가?
- [ ] 자바인데 Windows 에선 되고 Linux 에선 깨지는 코드 세 가지는?
- [ ] JVM 3대 구성요소와 각 역할을 댈 수 있는가?
- [ ] C/C++ 대비 자바가 치르는 대가 두 가지는?

## 관련

[[jdk-jre-jvm]] · [[interpreter-and-jit]] · [[jvm-stack-and-heap]] · [[garbage-collection-reachability]] · [[externalized-configuration]]
