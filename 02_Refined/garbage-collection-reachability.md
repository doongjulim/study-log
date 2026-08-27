---
type: refined
slug: garbage-collection-reachability
tags: [GC, 가비지컬렉션, garbage-collection, 도달가능성, reachability, GC-Root, unreachable, 참조, OutOfMemoryError, OOM, 메모리누수, memory-leak, Stop-the-World, STW, 세대별GC, young, old, minor-GC, full-GC, G1, ZGC, 힙덤프]
topic: GC 가 무엇을 쓰레기로 판정하는가 — 예측이 아니라 GC Root 로부터의 도달 가능성
summary: GC 는 "안 쓸 것 같은 객체"를 예측하지 않고, GC Root 에서 참조를 따라가 도달할 수 없는 객체만 기계적으로 수거한다. 그래서 참조가 남아있으면 GC 가 있어도 OOM 이 나고, 수거 중에는 Stop-the-World 가 발생한다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/en/java/javase/17/gctuning/index.html
updated: 2026-08-25
---

# GC 와 도달 가능성(Reachability)

## 개념 정의

**GC 의 판정 기준은 단 하나: GC Root 에서 참조를 따라가 도달할 수 있는가.**

- **GC Root**: 탐색의 출발점. 실행 중인 스레드의 **스택에 있는 지역변수·매개변수**,
  static 필드, JNI 참조 등.
- **도달 가능(reachable)** = 살아있는 객체 → 수거하지 않는다
- **도달 불가(unreachable)** = 쓰레기 → 수거 대상

```
[GC Root: 스택의 지역변수] ──> 객체 A ──> 객체 B      ← 둘 다 살아있음
                              객체 C                  ← 아무도 안 가리킴 = 수거 대상
```

**"재사용될 것 같지 않은 객체"를 예측하는 것이 아니다.** 순수하게 기계적인 도달 가능성 검사다.
그래서 A 와 B 가 서로만 가리키는 순환 참조라도, GC Root 에서 닿지 않으면 함께 수거된다.

## 작동 원리

### GC 가 있는데 왜 `OutOfMemoryError` 가 나는가

```java
List<byte[]> list = new ArrayList<>();
while (true) {
    list.add(new byte[1024 * 1024]);   // 계속 담기만 한다
}
```

`list` 가 계속 참조를 붙들고 있으므로 담긴 객체 전부가 **GC Root 에서 도달 가능**하다.
GC 는 이들을 "살아있는 객체"로 판단해 치우지 못한다. 결국 **살아있는 객체만으로 힙이 차서 OOM**.

> **메모리 누수의 자바식 정의**: "해제를 안 해서"가 아니라 **"더 이상 쓰지 않는데 참조가 남아있어서"** 다.
> 정적 컬렉션, 캐시, 리스너 등록 해제 누락, 장시간 유지되는 맵이 전형적인 범인이다.
> 대량 조회로 힙을 채우는 것도 같은 원리 ([[db-pagination]]).

### Stop-the-World

GC 가 힙을 정리하는 동안 **애플리케이션 스레드가 멈춘다.**
청소 중에도 객체 참조가 계속 바뀌면 정확한 수거가 불가능하기 때문이다.

이것이 게임 서버·초저지연 시스템에서 자바가 불리하다고 지적받는 **GC 의 대가**다.
GC 알고리즘의 역사는 사실상 **STW 시간을 줄이는 방향의 진화**다 (Serial → Parallel → G1 → ZGC).

### 세대별 구조는 별개의 축

힙을 young(eden/survivor) 과 old 로 나누는 세대별 GC 는
**"어디를 언제 청소하나"(장소·타이밍)의 최적화**다 — *"대부분의 객체는 금방 죽는다"* 는 경험칙을 이용한 것.
**쓰레기 판정 기준(도달 가능성)과는 다른 축의 이야기**다. 이 둘을 섞으면 개념이 무너진다.

## 트레이드오프 / 한계

- **`StackOverflowError` 와 헷갈리지 말 것** — 이건 힙이 아니라 스택 고갈이며 GC 와 무관하다
  ([[jvm-stack-and-heap]]).
- 힙을 크게 잡으면 GC 빈도는 줄지만 **한 번의 STW 가 길어진다.**
- GC 는 개발자의 메모리 해제 부담을 없애 주는 대신 **정지 시간과 CPU 를 가져간다.** 공짜가 아니다.
- OOM 진단은 추측이 아니라 **힙 덤프 분석**(`-XX:+HeapDumpOnOutOfMemoryError` → MAT 등)으로 한다.
  "무엇이 참조를 붙들고 있는가"를 찾는 작업이다.

> [!WARNING]
> **오답 코너**
> - **"GC 는 재사용되지 않을 객체를 지운다"** — 예측이 아니라 **도달 가능성(reachability)** 판정이다.
> - **"GC 는 young 영역의 객체를 지운다"** — young/old 는 **어디를 언제 청소하나**의 문제이지
>   쓰레기 판정 기준이 아니다. 두 개념을 섞지 말 것.
> - **"GC 가 있으니 메모리 누수는 없다"** — 참조가 남아있으면 GC 는 손대지 않는다. 누수는 존재한다.
> - **"`null` 을 대입하면 즉시 수거된다"** — 도달 불가 상태가 될 뿐, 수거 시점은 GC 가 정한다.
> - **"순환 참조는 GC 가 못 치운다"** — 참조 카운팅 방식의 이야기다. 도달 가능성 방식은 치운다.

## 복습 체크

- [ ] GC Root 가 무엇이고 어떤 것들이 여기에 해당하는가?
- [ ] 쓰레기 판정 기준을 한 문장으로 말할 수 있는가?
- [ ] GC 가 있는데도 OOM 이 나는 상황을 코드 수준으로 설명할 수 있는가?
- [ ] 자바에서 "메모리 누수"의 정의는?
- [ ] Stop-the-World 가 왜 필요한가?
- [ ] 세대별 GC 와 도달 가능성이 서로 다른 축이라는 것을 설명할 수 있는가?
- [ ] `StackOverflowError` 와 `OutOfMemoryError` 를 구분할 수 있는가?

## 관련

[[jvm-stack-and-heap]] · [[jvm-execution-pipeline]] · [[db-pagination]]
