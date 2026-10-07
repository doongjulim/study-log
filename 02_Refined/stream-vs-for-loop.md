---
type: refined
slug: stream-vs-for-loop
tags: [Stream, 스트림, Stream-API, for문, for-loop, 스트림vs포문, 언제스트림, when-to-use-stream, 쓰지말아야할때, 선언형, declarative, 명령형, imperative, 가독성, readability, 가변상태, mutable-state, 부수효과, side-effect, collect, toList, forEach, 결과값vs행위, 면접답변, 함수형프로그래밍, functional-programming]
topic: Stream 은 "결과값을 얻는 변환"에 쓰고, "행위가 목적인 반복"·checked 예외 I/O·병렬 함정 구간에는 for 문을 쓴다
summary: Stream 의 "가독성"은 구체적으로 명령형(How)이 아닌 선언형(What)이라 의도가 단어로 드러나고, 결과를 루프 도중 변하는 가변 변수에 쌓지 않는다는 뜻이다. 그래서 컬렉션을 변환·필터·집계해 결과값을 얻는 곳에 맞고, 결과가 아닌 부수효과가 목적인 반복, 람다 안 checked 예외가 많은 I/O, 바깥 상태를 수정하는 코드, 손해 보는 parallel 구간에는 for 문이 낫다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/package-summary.html
updated: 2026-10-07
---

# Stream vs for 문 — 어디에 쓰고 어디에 안 쓰나

## 개념 정의

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

"가독성이 좋다"를 구체적으로 풀면 두 가지다.

1. **의도가 단어로 드러난다.** (A) 는 읽는 사람이 루프를 머릿속으로 **직접 실행**해 봐야 "공개 글 제목 목록이구나"를 안다. (B) 는 `filter(공개) → map(제목) → toList` 만 읽으면 된다. SQL `WHERE` 가 인덱스 선택을 DB 에 맡기듯, 실행 방식은 라이브러리에 맡긴다.
2. **가변 중간 상태가 없다.** (A) 의 `titles` 는 빈 채로 태어나 **루프 도중 계속 변한다.** (B) 는 완성된 결과가 한 번에 대입된다. 이 차이가 병렬에서 갱신 유실 여부를 가른다 ([[race-condition-lost-update]]).

## 작동 원리 — 판단 기준

> Stream 은 **"데이터를 변환해서 결과를 얻는"** 도구다. 컬렉션 → 새 컬렉션 / 값.
> 결과가 아니라 **행위(부수효과)** 가 목적이면 선언형·지연·단락([[stream-lazy-evaluation]])이 쓸 데가 없다.

| ✅ Stream | ❌ for 문 |
|---|---|
| 컬렉션을 변환·필터·집계해 **결과값**을 얻을 때 | 결과가 아니라 **행위(부수효과)** 가 목적인 반복 (고정 N회 실행, 알림 전송) |
| 조건 맞는 첫 원소 (`filter` + `findFirst` = `break`) | 람다 안 **checked 예외**가 많은 I/O 로직 ([[lambda-checked-exception]]) |
| 큰 데이터 + CPU 연산 + ArrayList/배열 소스의 병렬 | `parallel()` + 블로킹 I/O / 작은 데이터 / LinkedList 소스 ([[parallel-stream-pitfalls]]) |
| | 람다 안에서 **바깥 변수·리스트를 수정**해야 할 때 |

- 고정 10번 반복이 for 문이 나은 이유는 성능(무시할 수준)이 아니라 **목적이 행위**이기 때문이다. `IntStream.range(0, 10).forEach(...)` 도 동작은 한다.
- `break` 는 단락 연산으로 표현할 수 있다:

```java
paths.stream()
     .filter(this::containsError)
     .findFirst()
     .ifPresent(this::alert);
```

## 트레이드오프 / 한계

- **Stream 이 안전한 게 아니라 공유 상태를 안 만드는 방식이 안전한 것이다.** Stream 안에서 바깥 리스트에 넣으면 for 문과 똑같은 버그가 난다:

```java
List<String> titles = new ArrayList<>();
posts.parallelStream()
     .filter(Post::isPublic)
     .forEach(p -> titles.add(p.getTitle()));   // ❌ 갱신 유실 — collect/toList 를 쓸 것
```

- `Stream.toList()`(Java 16+) 는 **수정 불가 리스트**를 돌려준다. 이후 add 가 필요하면 `collect(Collectors.toList())` 나 `new ArrayList<>(...)`.
- 디버깅: 람다 스택 트레이스가 깊고, 중간 값을 보기 어렵다 (`peek` 은 디버깅 용도로만).
- 박싱: `Stream<Integer>` 는 원소마다 박싱 → 숫자 연산은 `IntStream`/`mapToInt`.

### 면접 답변 뼈대

> "컬렉션을 **변환하거나 걸러서 결과값을 얻는 곳**에는 Stream 을 씁니다. 선언형이라 의도가 바로 읽히고, 중간 결과를 가변 리스트에 쌓지 않아 실수 여지가 적습니다.
> 반대로 **I/O 처럼 checked 예외가 나는 작업**이 중심이거나, 결과가 아니라 **동작 자체가 목적인 반복**에는 for 문을 씁니다. 람다 안에서 예외를 감싸다 보면 오히려 읽기 어려워지기 때문입니다.
> `parallel()` 은 데이터가 크고 CPU 연산 위주일 때만 고려합니다. 공용 ForkJoinPool 을 쓰기 때문에 블로킹 I/O 를 넣으면 다른 작업까지 막힐 수 있어서요."

> [!WARNING]
> **오답 코너**
> - **"Stream 은 람다라서 가독성이 좋다"** — 면접에선 "구체적으로?" 가 따라온다. **선언형(What)** 이라 의도가 드러나고 **가변 중간 상태가 없다**까지 말해야 한다.
> - **"for 문은 절차지향형, Stream 은 객체지향형"** — 그건 코드 조직 단위(함수 vs 객체)의 축이다. 이 비교의 축은 **명령형 vs 선언형**이다.
> - **"고정 횟수 반복은 성능 때문에 for 문"** — 10번이면 차이는 무시할 수준이다. 이유는 **목적이 결과가 아니라 행위**라서다.
> - **"Stream 을 쓰면 스레드 안전하다"** — `forEach` 로 바깥 상태를 바꾸면 똑같이 깨진다.

## 복습 체크

- [ ] "가독성이 좋다"를 명령형/선언형과 가변 상태 두 단어로 구체화할 수 있는가?
- [ ] 명령형/선언형 축과 절차지향/객체지향 축의 차이는?
- [ ] Stream 이 맞는 곳과 for 문이 나은 곳을 각각 2개 이상 근거와 함께 말할 수 있는가?
- [ ] `break` 를 Stream 으로 어떻게 표현하는가?
- [ ] "Stream 이 안전한 게 아니다"의 의미를 `forEach(list::add)` 예로 설명할 수 있는가?
- [ ] 면접 답변 뼈대에 내 프로젝트 장면 하나를 붙여 말할 수 있는가?

## 관련

[[stream-lazy-evaluation]] · [[race-condition-lost-update]] · [[lambda-checked-exception]] · [[parallel-stream-pitfalls]] · [[java-optional]]
