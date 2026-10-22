---
type: refined
slug: jdk-jre-jvm
tags: [JDK, JRE, JVM, Java-Development-Kit, Java-Runtime-Environment, 자바가상머신, 자바개발도구, 실행환경, runtime-environment, 포함관계, 표준클래스라이브러리, standard-class-library, java.lang, java.util, javac, jar, javadoc, jdb, 개발도구, development-tools, 운영서버, jlink, 커스텀런타임, custom-runtime, eclipse-temurin, Adoptium, 멀티스테이지빌드, multi-stage-build, Docker, 면접답변]
topic: "JDK ⊃ JRE ⊃ JVM — 실행에 필요한 것과 개발에 필요한 것을 나눈 포함 관계"
summary: "JVM 은 바이트코드를 실행하는 엔진, JRE 는 JVM 에 String·ArrayList 같은 표준 클래스 라이브러리를 더한 실행 환경, JDK 는 JRE 에 javac·jar·jdb 같은 개발 도구를 더한 것이다. 개발 PC 는 컴파일·테스트를 하니 JDK 가, 운영 서버는 실행만 하니 JRE 가 필요하다. Java 11 부터 오라클은 별도 JRE 배포를 중단했지만 jlink 커스텀 런타임과 JRE 전용 Docker 이미지로 '빌드는 JDK, 실행은 JRE' 원리는 그대로 이어진다."
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/en/java/javase/17/install/overview-jdk-installation.html
  - https://docs.oracle.com/en/java/javase/17/docs/specs/man/jlink.html
updated: 2026-10-22
---

# JDK · JRE · JVM

## 개념 정의

```
┌──────────────── JDK ────────────────────────────┐
│  개발 도구: javac, jar, javadoc, jdb …          │
│  ┌──────────── JRE ──────────────────────────┐  │
│  │  표준 클래스 라이브러리                    │  │
│  │  (java.lang.String, java.util.ArrayList,   │  │
│  │   java.lang.System …)                      │  │
│  │  ┌──────── JVM ────────┐                   │  │
│  │  │ 바이트코드 실행 엔진 │                   │  │
│  │  └─────────────────────┘                   │  │
│  └────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┘
```

| 구성 | 공식 | 역할 | 필요한 곳 |
|---|---|---|---|
| **JVM** | — | `.class` 바이트코드를 실행하는 엔진 | — |
| **JRE** | JVM + 표준 클래스 라이브러리 | 자바 프로그램을 **실행**하는 데 필요한 최소 환경 | 운영 서버 |
| **JDK** | JRE + 개발 도구 | 자바 프로그램을 **만들고** 실행 | 개발 PC, CI 빌드 |

JDK 는 Java **Development** Kit 다 (Developer 아님).

## 작동 원리

### JRE 에 왜 라이브러리가 들어 있나

`String`, `ArrayList`, `System.out` 은 내가 작성한 코드가 아닌데도 **실행 시점에 실제 구현 코드로 존재해야** 한다.
JVM 은 실행기일 뿐 이 클래스들을 갖고 있지 않다. 그래서 실행 환경(JRE)은 **실행기 + 실행기가 불러 쓸 표준 라이브러리**의 묶음이다.

운영체제는 JRE 구성요소가 아니다. 원래 컴퓨터에 깔려 있는 바탕이고, JRE 는 그 위에 올라간다.

### 왜 포함 관계가 이 방향인가

운영 서버는 이미 컴파일된 `.class`/`.jar` 를 **실행만** 한다. 컴파일러(`javac`)가 필요 없다.
따라서 실행 환경(JRE)이 개발 도구(JDK)를 품을 이유가 없고, 반대로 개발하는 쪽은 실행도 해 봐야 하므로 JDK 가 JRE 를 품는다.

## 트레이드오프 / 실무

- **Java 11 부터 오라클은 별도 JRE 배포를 중단**했다. 대신 `jlink` 로 앱이 쓰는 모듈만 담은 **커스텀 런타임**을 만들 수 있다 → 이미지 크기·공격 표면 축소.
- Eclipse Temurin(Adoptium) 등은 여전히 `eclipse-temurin:21-jre` 같은 **JRE 전용 이미지**를 제공한다.
- 그래서 컨테이너 배포는 **멀티스테이지 빌드**가 흔하다.

```dockerfile
FROM eclipse-temurin:21-jdk AS build      # 빌드 단계: JDK (javac, gradle)
WORKDIR /app
COPY . .
RUN ./gradlew bootJar

FROM eclipse-temurin:21-jre               # 실행 단계: JRE 만
COPY --from=build /app/build/libs/app.jar /app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

"개발은 JDK, 운영은 JRE" 원리가 컨테이너 시대에도 그대로 살아 있다.

> [!WARNING]
> **오답 코너**
> - **"JRE = JVM + JDK"** — 포함 관계가 뒤집혔다. 그러면 실행만 하는 운영 서버에 컴파일러가 들어 있어야 한다. **JDK ⊃ JRE ⊃ JVM.**
> - **"JDK = JAVA + JAVAC"** — JDK 는 JRE 전체(JVM + 표준 라이브러리) **위에** 개발 도구 묶음이 얹힌 것이다. javac 는 그 도구 중 하나.
> - **"JRE 에 필요한 건 JVM + 운영체제"** — OS 는 바탕이지 JRE 구성요소가 아니다. 빠진 조각은 **표준 클래스 라이브러리**.
> - **"JRE 는 가장 표준적인 자바 기능의 집합체"** — 추상적이다. "실행기(JVM) + 실행기가 불러 쓰는 표준 라이브러리"로 구체화할 것.
> - **"JDK = Java Developer Kit"** — **Development** Kit.

## 면접 답변

> "JVM 은 바이트코드를 실행하는 엔진이고, JRE 는 거기에 String 이나 ArrayList 같은 표준 클래스 라이브러리를 더한 실행 환경, JDK 는 JRE 에 javac 같은 개발 도구를 더한 것입니다. 그래서 개발 PC 에는 JDK 가, 실행만 하는 운영 서버에는 JRE 면 충분합니다. 실무에서는 Docker 멀티스테이지 빌드로 빌드 단계는 JDK 이미지, 실행 단계는 JRE 이미지를 써서 이미지 크기를 줄입니다."

## 복습 체크

- [ ] JDK / JRE / JVM 의 포함 관계를 그림으로 그리고, 각각의 공식(?? + ??)을 말할 수 있는가?
- [ ] JRE 에 표준 클래스 라이브러리가 들어 있어야 하는 이유는?
- [ ] 개발 PC 와 운영 서버에 각각 무엇이 필요하고, 그 이유는?
- [ ] "JRE 안에 JDK" 라고 하면 어떤 모순이 생기는가?
- [ ] Java 11 이후 JRE 가 사라졌다는데, 운영 이미지는 어떻게 가볍게 만드는가?

## 관련

[[jvm-execution-pipeline]] · [[interpreter-and-jit]] · [[jvm-container-heap-sizing]]
