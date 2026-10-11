---
date: 2026-10-11
type: raw
author: dongju
tags: [JVM 옵션, JVM options, 힙, heap, 최대 힙, max heap, Xmx, Xms, MaxRAMPercentage, InitialRAMPercentage, 컨테이너, container, Docker, Dockerfile, cgroup, 컨테이너 인식, container awareness, OOMKilled, exit code 137, SIGKILL, OutOfMemoryError, Java heap space, HeapDumpOnOutOfMemoryError, 힙 밖 메모리, off-heap, non-heap, Metaspace, 스레드 스택, thread stack, Code Cache, Direct Memory, Serial GC, G1, GC ergonomics, eclipse-temurin, 면접 답변, study-board]
topic: 컨테이너에서 -Xmx 고정값 대신 -XX:MaxRAMPercentage=75 를 준 이유 — 기본 25% 의 낭비, 100% 의 위험(힙 밖 메모리), 고정값의 OOMKilled
summary: MaxRAMPercentage 는 cgroup 으로 읽은 컨테이너 메모리 한도의 비율로 최대 힙을 정한다. 옵션이 없으면 기본 25% 라 1GB 컨테이너에서 힙은 약 256MB 이고 나머지 약 750MB 가 논다. 100% 면 Metaspace·스레드 스택·Code Cache·Direct Memory 같은 힙 밖 영역이 들어갈 자리가 없어 커널에 OOMKilled 된다. 75% 는 그 여유분을 남긴 값이다. -Xmx768m 고정값은 운영에서 한도를 512MB 로 낮추면 힙이 한도를 넘어 흔적 없는 OOMKilled(137)로 죽지만, 퍼센트는 한도를 따라 힙이 같이 줄어 최악에도 원인을 볼 수 있는 OutOfMemoryError 로 실패한다.
source: session
distilled: true
distilled_at: 2026-10-11
---

## 배운 개념

### 1. 실제 설정 (study-board Dockerfile)

```dockerfile
FROM eclipse-temurin:17-jre AS runtime
...
# 컨테이너 메모리에 맞춰 힙을 잡게 한다 (고정값을 박으면 한도를 바꿔도 따라오지 않는다)
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "-jar", "app.jar"]
```

- 75% 의 기준 = **컨테이너에 걸린 메모리 한도**. JDK 10+ (8u191 백포트) 부터 JVM 이 **cgroup** 을 읽어 호스트가 아니라 컨테이너 한도로 계산한다.
- 참고: `gradlew` 의 `DEFAULT_JVM_OPTS='"-Xmx64m" "-Xms64m"'` 는 **Gradle 실행기** 설정 → 애플리케이션 힙과 무관 (코드 검색 시 헷갈리기 쉬움).

### 2. 왜 100% 가 아닌가 — 힙 밖 메모리

```
컨테이너 한도 1GB
└─ 힙 1GB (100%)
   └─ 그럼 이것들은 어디에?
      - 로드된 클래스 정보 (Metaspace)
      - 스레드마다 하나씩 있는 스택 (요청 스레드 200개 × 약 1MB)
      - JIT 이 컴파일한 기계어 (Code Cache)
      - GC 내부 자료구조, 네트워크 버퍼 (Direct Memory)
```

JVM 프로세스는 힙 말고도 메모리를 꽤 쓴다 → 힙을 한도만큼 잡으면 한도 초과 → 커널이 **OOMKilled**.

### 3. 기본값 25% vs 75%

| 설정 | 1GB 컨테이너 최대 힙 | 결과 |
|---|---|---|
| 옵션 없음 (기본 **25%**) | 약 256MB | 힙이 차면 GC 반복 → `OutOfMemoryError: Java heap space`. 그동안 약 750MB 는 돈 내고 비워 둠 |
| `MaxRAMPercentage=75` | 약 768MB | 나머지 약 256MB 를 힙 밖 영역 몫으로 |
| (가정) 100% | 1GB | 힙 밖 영역 자리 없음 → OOMKilled |

### 4. 퍼센트 vs 고정값 — "어떻게 죽느냐"

```dockerfile
ENTRYPOINT ["java", "-Xmx768m", "-jar", "app.jar"]
```

1GB 에서 잘 돌다가 운영팀이 비용 절감으로 한도를 **512MB** 로 낮추면:

- JVM 은 여전히 힙을 **768MB 까지** 키우려 한다 → 한도 초과 → **OOMKilled**.
- 퍼센트였다면 힙이 384MB(75%)로 **자동으로 따라온다**.

| | `OutOfMemoryError` | OOMKilled |
|---|---|---|
| 누가 | JVM | 리눅스 커널 (`SIGKILL`) |
| 흔적 | 스택 트레이스, `-XX:+HeapDumpOnOutOfMemoryError` 로 힙 덤프 | 거의 없음, 종료 코드 **137**, shutdown hook 도 안 돎 |
| 겉보기 | 에러 로그 | "가끔 이유 없이 재시작되는 서버", 처리 중 요청 끊김 |

→ 퍼센트는 최악에도 **원인이 보이는 쪽으로** 실패하게 만든다.

### 5. 현재 설정의 남은 개선점

- `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/app/data` — OOM 원인 분석 자료 (볼륨 경로에).
- `-XX:InitialRAMPercentage` — 초기 힙 기본값이 매우 작아(약 1.6%) 기동 직후 힙 확장 중 GC 가 잦을 수 있다.
- 작은 컨테이너에서 75% — 256MB 한도면 힙 밖 몫이 64MB 뿐. 한도가 작을수록 비율을 낮추는 게 안전.
- GC 자동 선택 — JDK 17 은 CPU 2개 미만이거나 메모리 약 1792MB 미만이면 G1 대신 **Serial GC** 를 고른다. 1GB 컨테이너면 실제 선택 확인 필요.

```bash
java -XX:MaxRAMPercentage=75 -XX:+PrintFlagsFinal -version | grep -E "MaxHeapSize|UseSerialGC|UseG1GC"
```

### 6. 면접 답변 뼈대 (JVM 파트)

> "JVM 옵션은 Docker 이미지에서 `-Xmx` 고정값 대신 `MaxRAMPercentage=75` 를 줬습니다. 기본값 25% 는 컨테이너 메모리의 4분의 3을 놀리고, 고정값은 운영에서 한도를 줄이면 힙이 한도를 넘어 커널에 OOMKilled 로 흔적 없이 죽습니다. 75% 면 한도를 따라 힙이 같이 움직이고, 나머지 25% 는 Metaspace 나 스레드 스택 같은 힙 밖 영역의 몫으로 남겨 둔 겁니다."

## 오답·헷갈린 점

- **전**: 옵션을 안 주면 JVM 이 메모리를 "**항상 100**" 으로 잡는다 → **후**: 기본 `MaxRAMPercentage` 는 **25%** 로 보수적. 100% 면 힙 밖 영역 자리가 없어 OOMKilled.
- **전**: 1GB 의 25% = "250gb" → **후**: 오타. **약 256MB**.
- **정답 맞힘**: 75% 의 기준은 컨테이너 할당 메모리 / 75 에서 멈춘 이유는 힙 밖 여유분 / 고정 -Xmx768m + 한도 512MB → 768 까지 키우려다 OOMKilled, 운영에서 잡기 어려움.

## Q&A

- **Q. MaxRAMPercentage=75 의 75% 는 무엇의?** A. cgroup 으로 읽은 컨테이너 메모리 한도.
- **Q. 옵션이 없으면?** A. 기본 25%. 1GB 면 약 256MB.
- **Q. 왜 100 이 아니라 75?** A. 힙 밖(Metaspace, 스레드 스택, Code Cache, Direct Memory) 여유분.
- **Q. 왜 -Xmx 고정값이 아니라 퍼센트?** A. 한도가 바뀌면 힙이 따라온다. 고정값은 한도를 줄이면 OOMKilled(137, 흔적 없음).

## 다음 학습

- JVM 메모리 영역 전체 그림: Metaspace, Code Cache, Direct Memory, `-Xss`
- 컨테이너 안에서 실제 선택된 GC·힙 확인, Serial vs G1 선택 기준
- Native Memory Tracking (`-XX:NativeMemoryTracking=summary`, `jcmd VM.native_memory`)
