---
type: refined
slug: jvm-execution-pipeline
tags: [JVM, 자바가상머신, 실행파이프라인, javac, 컴파일, 바이트코드, bytecode, class파일, WORA, Write-Once-Run-Anywhere, 플랫폼독립성, 클래스로더, class-loader, 런타임데이터영역, runtime-data-area, 실행엔진, execution-engine, JRE, JDK]
topic: .java 가 실행되기까지의 경로와 JVM 3대 구성요소, WORA 가 성립하는 구조
summary: javac 가 소스를 JVM 전용 중간 코드인 바이트코드(.class)로 컴파일하고, OS 별로 따로 구현된 JVM 이 이를 읽어 실행한다. 플랫폼 차이를 JVM 이 흡수하는 구조가 WORA 의 실체다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/javase/specs/jvms/se17/html/index.html
updated: 2026-08-25
---

# JVM 실행 파이프라인과 WORA

## 개념 정의

```
Hello.java  --[javac]-->  Hello.class  --[JVM]-->  실행
(소스코드)   (컴파일러)     (바이트코드)   (가상머신)
```

**바이트코드는 기계어도 소스코드도 아닌 JVM 전용 중간 코드**다. OS 가 아니라 JVM 이 읽는다.

## 작동 원리

### WORA 가 성립하는 구조

C/C++ 는 소스를 **타깃 기계어로 직접** 컴파일한다. 그래서 OS·CPU 아키텍처가 바뀌면 다시 빌드해야 한다.
이것이 자바가 해결하려 한 Pain Point다.

자바는 층을 하나 끼워 넣는다.

```
어느 플랫폼에서든 동일한 Hello.class (바이트코드)
        ↓            ↓             ↓
   Windows용 JVM  macOS용 JVM   Linux용 JVM     ← 플랫폼 차이를 여기서 흡수
        ↓            ↓             ↓
     각 OS의 기계어로 변환되어 실행
```

**바이트코드는 어디서나 같고, 다른 것은 JVM 쪽이다.**
"Write Once, Run Anywhere" 는 *"자바가 플랫폼을 안 탄다"* 가 아니라
*"플랫폼을 타는 부분을 JVM 이 대신 떠맡는다"* 는 뜻이다.

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
- **"어디서나 똑같이 동작"은 이상에 가깝다** — 파일 경로 구분자, 기본 문자 인코딩, 스레드 스케줄링,
  JVM 벤더·버전 차이에서 플랫폼 의존성이 새어 나온다.
- 바이트코드는 사람이 읽기 쉬운 구조라 **역컴파일에 취약**하다(난독화가 별도 과제가 되는 이유).

> [!WARNING]
> **오답 코너**
> - **"javac 파일로 컴파일된다"** — `javac` 는 **컴파일러(명령어) 자체**이고,
>   산출물은 **`.class` 파일(바이트코드)** 이다. 도구 이름과 결과물을 혼동하기 쉽다.
> - **"바이트코드는 기계어다"** — 아니다. CPU 가 아니라 **JVM 이 읽는 중간 코드**다.
> - **"자바는 인터프리터 언어다"** — 컴파일(javac) + 인터프리트 + JIT 컴파일이 모두 섞인 하이브리드다.
> - **"WORA 는 자바 언어의 성질"** — **JVM 이 OS 별로 따로 구현되어 있기 때문에** 성립하는 것이다.

## 복습 체크

- [ ] `.java` 부터 실행까지의 단계를 도구 이름과 산출물까지 정확히 말할 수 있는가?
- [ ] 바이트코드가 무엇이고 누가 읽는지 설명할 수 있는가?
- [ ] WORA 가 성립하는 이유를 "JVM 이 OS 별로 따로 있다"로 설명할 수 있는가?
- [ ] JVM 3대 구성요소와 각 역할을 댈 수 있는가?
- [ ] C/C++ 대비 자바가 치르는 대가 두 가지는?

## 관련

[[interpreter-and-jit]] · [[jvm-stack-and-heap]] · [[garbage-collection-reachability]]
