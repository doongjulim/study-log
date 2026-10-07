---
date: 2026-10-07
type: raw
author: dongju
tags: [Stream, 스트림, Stream API, 선언형, declarative, 명령형, imperative, 가독성, readability, 가변 상태, mutable state, 공유 가변 상태, shared mutable state, 경쟁 조건, race condition, 갱신 유실, lost update, 스레드 안전, thread safety, parallel, 병렬 스트림, collect, toList, forEach 부수효과, side effect, for문 vs 스트림]
topic: Stream 이 for 문 대비 해결하는 것 — 선언형 기술과 공유 가변 상태 제거 (병렬 add 시 갱신 유실)
summary: Stream 의 "가독성"은 구체적으로 명령형(How)이 아닌 선언형(What)이라 의도가 단어로 드러난다는 뜻이고, 중간 결과를 루프 도중 변하는 가변 리스트에 쌓지 않는다는 점이 핵심이다. 공유 ArrayList 에 여러 스레드가 add 하면 같은 size 를 읽고 같은 칸을 덮어써 갱신이 유실되지만, collect/toList 는 스레드별로 따로 모아 합치므로 구조적으로 안전하다. 단 Stream 안에서 바깥 리스트를 건드리면 같은 버그가 재발한다.
source: session
distilled: false
---

## 배운 개념

### 1. 같은 결과, 다른 스타일

```java
// (A) for문 — 명령형
List<String> titles = new ArrayList<>();
for (Post post : posts) {
    if (post.isPublic()) {
        titles.add(post.getTitle());
    }
}

// (B) Stream — 선언형
List<String> titles = posts.stream()
        .filter(Post::isPublic)
        .map(Post::getTitle)
        .toList();
```

| 스타일 | 이름 | 비유 |
|---|---|---|
| (A) 단계별 지시 (How) | **명령형** Imperative | 택시 기사에게 "직진, 두 번째 골목 좌회전, 100m 가서 정차" |
| (B) 원하는 결과 기술 (What) | **선언형** Declarative | "강남역 가 주세요" |

- "가독성이 좋다"의 구체적 의미: (A) 는 읽는 사람이 머릿속으로 루프를 **직접 실행**해야 결과를 알 수 있고, (B) 는 `filter(공개) → map(제목) → toList` 단어만 읽으면 된다.
- SQL 이 대표적 선언형 — `WHERE` 만 쓰고 인덱스/풀스캔은 DB 가 결정. Stream 도 같은 발상.
- 절차지향 vs 객체지향 은 **코드 조직 단위**(함수 vs 객체)의 축 → 지금 축(로직 기술 방식)과 다르다.

### 2. 가변 중간 상태의 위험 — 경쟁 조건

(A) 의 `titles` 는 빈 상태로 태어나 **루프 도중 계속 변한다.** 이 루프를 4 스레드로 나눠 같은 `titles` 에 add 하면:

`ArrayList.add()` = `elementData[size] = e` + `size++` 두 단계. `size = 5` 에서:

```
         스레드 1 (제목 "A")              스레드 2 (제목 "B")
 t1   size 읽음 → 5
 t2                                    size 읽음 → 5
 t3   elementData[5] = "A"
 t4                                    elementData[5] = "B"
 t5   size = 5 + 1 → 6
 t6                                    size = 5 + 1 → 6
```

결과: `elementData[5] = "B"`, **"A" 는 사라짐**, size 는 6 (add 2번이니 7 이어야 함).

- 이 현상 = **경쟁 조건(Race Condition)**, 결과 = **갱신 유실(Lost Update)**. 에러 없이 데이터만 조용히 사라진다.
- 확장(grow) 순간에 겹치면 `ArrayIndexOutOfBoundsException` 도 가능.

### 3. Stream 은 왜 안전한가 — 그리고 언제 안전하지 않은가

- `collect()` / `toList()` 는 공유 리스트에 add 하지 않는다. `parallel()` 이면 **스레드마다 자기 컨테이너**에 모은 뒤 마지막에 **합친다(combine).**
- ⚠️ 하지만 Stream 안에서 바깥 리스트를 직접 건드리면 똑같은 버그:

```java
List<String> titles = new ArrayList<>();
posts.parallelStream()
     .filter(Post::isPublic)
     .forEach(p -> titles.add(p.getTitle()));   // ❌ 공유 가변 상태 → 갱신 유실
```

→ **Stream 이 안전한 게 아니라, 공유 상태를 안 만드는 방식이 안전한 것.**

## 오답·헷갈린 점

- **전**: Stream 은 "람다식 표현으로 가독성이 좋다" → **후**: 면접에선 "구체적으로?" 가 따라온다. **선언형(What)** 이라 의도가 드러나고 **가변 중간 상태가 없다**.
- **전**: (A) 는 절차지향형, (B) 는 **객체지향형** 프로그래밍 → **후**: 그건 조직 단위의 축. 이 비교의 축은 **명령형 vs 선언형**.
- **전**: 공유 리스트 병렬 add 시 "**순서가 꼬인다**" → **후**: 같은 size 를 읽어 같은 칸을 덮어쓴다 → **갱신 유실**(데이터 소실 + size 불일치). 타임라인을 따라가며 스스로 도출.

## Q&A

- **Q. 자바 8 은 왜 Stream 을 추가했나?**
  A. 명령형 루프는 "어떻게" 를 단계별로 지시해 의도가 묻히고, 결과를 가변 변수에 쌓는다. Stream 은 "무엇을" 선언하고 결과를 한 번에 만든다.
- **Q. 4 스레드가 같은 ArrayList 에 add 하면?**
  A. 경쟁 조건으로 갱신 유실. 같은 칸 덮어쓰기, size 불일치, 확장 중 AIOOBE.
- **Q. 병렬 Stream 의 collect 는 왜 괜찮나?**
  A. 스레드별 컨테이너에 따로 모으고 마지막에 combine — 공유 가변 상태가 없다.

## 다음 학습

- `Collector` 의 supplier / accumulator / combiner / finisher 4요소
- `Collections.synchronizedList` / `CopyOnWriteArrayList` vs collect — 언제 무엇을
- 원자성(`AtomicInteger`, CAS)과 `size++` 가 원자적이지 않은 이유
