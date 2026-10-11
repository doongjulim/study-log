---
type: refined
slug: jvm-container-heap-sizing
tags: [JVM옵션, JVM-options, 힙크기, heap-size, 최대힙, max-heap, Xmx, Xms, MaxRAMPercentage, InitialRAMPercentage, MinRAMPercentage, 컨테이너, container, Docker, Dockerfile, cgroup, 컨테이너인식, container-awareness, OOMKilled, exit-code-137, SIGKILL, OutOfMemoryError, Java-heap-space, HeapDumpOnOutOfMemoryError, 힙밖메모리, off-heap, non-heap, Metaspace, 스레드스택, thread-stack, Code-Cache, Direct-Memory, Serial-GC, G1, GC-ergonomics, PrintFlagsFinal, 면접답변]
topic: 컨테이너에서는 -Xmx 고정값 대신 -XX:MaxRAMPercentage 로 힙을 잡는다 — 기본 25% 의 낭비, 100% 의 위험(힙 밖 메모리), 고정값의 OOMKilled
summary: MaxRAMPercentage 는 cgroup 으로 읽은 컨테이너 메모리 한도의 비율로 최대 힙을 정한다. 기본값 25% 면 1GB 컨테이너의 힙이 약 256MB 라 나머지가 놀고, 100% 면 Metaspace·스레드 스택·Code Cache·Direct Memory 가 들어갈 자리가 없어 커널에 OOMKilled 된다. 75% 는 힙 밖 여유분을 남긴 값이다. -Xmx 고정값은 운영에서 한도를 줄이면 힙이 한도를 넘어 흔적 없는 OOMKilled(137)로 죽지만, 퍼센트는 한도를 따라 힙이 줄어 최악에도 원인을 볼 수 있는 OutOfMemoryError 로 실패한다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/en/java/javase/17/docs/specs/man/java.html
  - https://docs.oracle.com/en/java/javase/17/gctuning/ergonomics.html
updated: 2026-10-11
---

# 컨테이너 JVM 힙 크기 — MaxRAMPercentage

## 개념 정의

```dockerfile
FROM eclipse-temurin:17-jre
# 컨테이너 메모리에 맞춰 힙을 잡게 한다 (고정값을 박으면 한도를 바꿔도 따라오지 않는다)
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "-jar", "app.jar"]
```

- 기준은 **컨테이너에 걸린 메모리 한도**. JDK 10+ (8u191 백포트) 부터 JVM 이 **cgroup** 을 읽어 호스트가 아니라 컨테이너 한도로 계산한다.
- 옵션이 없으면 `MaxRAMPercentage` 기본값은 **25%** (아주 작은 메모리에선 별도 규칙).
- 혼동 주의: `gradlew` 의 `DEFAULT_JVM_OPTS='"-Xmx64m"'` 는 **Gradle 실행기** 설정이라 애플리케이션 힙과 무관하다.

## 작동 원리

### 힙 밖에서도 메모리를 쓴다

```
컨테이너 한도 1GB
└─ 힙 1GB (100%)
   └─ 그럼 이것들은 어디에?
      - 로드된 클래스 정보 (Metaspace)
      - 스레드마다 하나씩 있는 스택 (요청 스레드 200개 × 약 1MB)  ([[jvm-stack-and-heap]])
      - JIT 이 컴파일한 기계어 (Code Cache)  ([[interpreter-and-jit]])
      - GC 내부 자료구조, 네트워크 버퍼 (Direct Memory)
```

### 25% / 75% / 100%

| 설정 | 1GB 컨테이너 최대 힙 | 결과 |
|---|---|---|
| 옵션 없음 (25%) | 약 256MB | 힙이 차면 GC 반복 → `OutOfMemoryError`. 약 750MB 는 돈 내고 비워 둠 |
| `MaxRAMPercentage=75` | 약 768MB | 나머지 약 256MB 는 힙 밖 영역 몫 |
| (가정) 100% | 1GB | 힙 밖 영역 자리 없음 → **OOMKilled** |

### 퍼센트 vs 고정값 — "어떻게 죽느냐"

`-Xmx768m` 으로 1GB 에서 잘 돌다가, 운영팀이 비용 절감으로 한도를 **512MB** 로 낮추면:

- JVM 은 여전히 힙을 **768MB 까지** 키우려 한다 → 한도 초과 → **OOMKilled**.
- 퍼센트였다면 힙이 384MB(75%)로 **자동으로 따라온다**.

| | `OutOfMemoryError` | OOMKilled |
|---|---|---|
| 누가 | JVM | 리눅스 커널 (`SIGKILL`) |
| 흔적 | 스택 트레이스, 힙 덤프 가능 ([[garbage-collection-reachability]]) | 거의 없음, 종료 코드 **137**, shutdown hook 도 안 돎 |
| 겉보기 | 에러 로그 | "가끔 이유 없이 재시작되는 서버", 처리 중 요청 끊김 |

→ 퍼센트는 최악에도 **원인이 보이는 쪽으로** 실패하게 만든다.

## 트레이드오프 / 한계

- **작은 컨테이너에서 75% 는 빠듯하다.** 256MB 한도면 힙 밖 몫이 64MB 뿐이다. 한도가 작을수록 비율을 낮춘다.
- **초기 힙(`InitialRAMPercentage`) 기본값이 매우 작다**(약 1.6%). 기동 직후 힙을 늘리는 동안 GC 가 잦을 수 있다.
- **OOM 흔적을 남기려면** `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=<볼륨 경로>`.
- **GC 자동 선택**: JDK 17 은 CPU 2개 미만이거나 메모리 약 1792MB 미만이면 G1 대신 **Serial GC** 를 고른다. 작은 컨테이너에선 실제 선택을 확인한다.

```bash
java -XX:MaxRAMPercentage=75 -XX:+PrintFlagsFinal -version | grep -E "MaxHeapSize|UseSerialGC|UseG1GC"
```

### 면접 답변 뼈대

> "Docker 이미지에서 `-Xmx` 고정값 대신 `MaxRAMPercentage=75` 를 줬습니다. 기본값 25% 는 컨테이너 메모리의 4분의 3을 놀리고, 고정값은 운영에서 한도를 줄이면 힙이 한도를 넘어 커널에 OOMKilled 로 흔적 없이 죽습니다. 75% 면 한도를 따라 힙이 같이 움직이고, 나머지 25% 는 Metaspace 나 스레드 스택 같은 힙 밖 영역의 몫으로 남겨 둔 겁니다."

> [!WARNING]
> **오답 코너**
> - **"옵션을 안 주면 JVM 이 메모리를 100% 잡는다"** — 기본값은 **25%** 로 보수적이다. 100% 면 힙 밖 영역 자리가 없어 OOMKilled 다.
> - **"MaxRAMPercentage 는 호스트 메모리 기준"** — 컨테이너 한도(cgroup) 기준이다.
> - **"메모리가 부족하면 늘 OutOfMemoryError 가 난다"** — 힙이 한도를 넘으면 JVM 이 아니라 **커널**이 죽인다(137). 로그가 거의 없다.

## 복습 체크

- [ ] `MaxRAMPercentage=75` 의 75% 는 무엇의 비율인가?
- [ ] 옵션이 없을 때의 기본 비율과, 1GB 컨테이너에서의 최대 힙은?
- [ ] 100% 가 아니라 75% 인 이유를 힙 밖 영역 네 가지로 설명할 수 있는가?
- [ ] `-Xmx768m` + 한도 512MB 시나리오에서 무슨 일이 생기는가?
- [ ] OutOfMemoryError 와 OOMKilled 의 차이(누가, 흔적, 종료 코드)는?
- [ ] 현재 설정에 더할 만한 옵션 두 가지는?

## 관련

[[jvm-stack-and-heap]] · [[garbage-collection-reachability]] · [[interpreter-and-jit]] · [[java17-lts-features]]
