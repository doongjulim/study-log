---
type: refined
slug: jvm-stack-and-heap
tags: [스택, stack, 힙, heap, 런타임데이터영역, 메모리구조, 스택프레임, stack-frame, 지역변수, 기본형, 참조, reference, 참조변수, 스레드별, thread-private, 공유메모리, shared-memory, 스레드안전, thread-safety, 동시성, concurrency, StackOverflowError, 메서드영역, Error, Exception, Throwable, ExceptionHandler, 무한재귀, infinite-recursion]
topic: 스택은 스레드마다 하나, 힙은 전체가 공유하는 하나 — 이 비대칭이 자바 동시성 문제의 뿌리
summary: 기본형 지역변수와 참조는 스택에, new 로 만든 객체는 힙에 놓인다. 스택은 스레드마다 독립이라 지역변수는 태생적으로 스레드 안전하고, 힙은 공유되기에 객체 교환이 가능한 대신 동시성 문제가 발생한다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/javase/specs/jvms/se17/html/jvms-2.html
updated: 2026-10-09
---

# JVM 메모리 구조 — 스택과 힙

## 개념 정의

```java
public void order() {
    int count = 3;
    Pizza p = new Pizza("페퍼로니");
}
```

| 대상 | 위치 |
|---|---|
| `count` (기본형 지역변수) | **스택** |
| `new Pizza(...)` 로 만든 객체 | **힙** |
| `p` (참조 변수) | **스택** — 안에는 객체 자체가 아니라 **힙 객체의 주소(참조)** |

메서드가 끝나면 스택 프레임이 통째로 제거되며 `count` / `p` 도 함께 사라진다.
힙의 `Pizza` 객체는 남고, 그 처리는 GC 의 몫이다 ([[garbage-collection-reachability]]).

## 작동 원리

### 결정적 차이 — 개수

| | 스택 | 힙 |
|---|---|---|
| 개수 | **스레드마다 1개씩** (thread-private) | **전체 스레드가 공유하는 단 1개** |
| 담기는 것 | 스택 프레임(지역변수, 매개변수, 반환 주소) | 모든 객체 인스턴스, 배열 |
| 정리 | 메서드 종료 시 프레임 제거 (자동) | GC |
| 고갈 시 | `StackOverflowError` | `OutOfMemoryError` |

**스레드가 3개면 스택도 3개, 힙은 여전히 1개다.**

### 이 비대칭이 만드는 두 결과

**① 지역변수는 태생적으로 스레드 안전하다.**
스택은 스레드마다 독립적으로 존재하므로, 다른 스레드가 내 지역변수에 접근할 **경로 자체가 없다.**
동기화가 필요 없는 이유가 "조심해서 짜서"가 아니라 **구조적으로 도달 불가능하기 때문**이다.

**② 힙 공유가 자바 동시성 문제의 구조적 뿌리다.**
힙이 하나이기에 스레드끼리 객체를 주고받을 수 있다. 만약 힙도 스레드별로 분리되면
스레드 A 가 만든 객체를 B 가 영영 볼 수 없어 **객체 공유 자체가 불가능**해진다.
그 대신 여러 스레드가 같은 객체를 동시에 건드릴 수 있고, 여기서 경쟁 조건이 발생한다 ([[race-condition-lost-update]]).

> 싱글턴 빈(스프링의 기본 스코프)에 가변 필드를 두면 위험한 이유가 정확히 이것이다.
> 빈은 힙에 하나뿐이고 모든 요청 스레드가 그것을 공유한다 ([[spring-data-repository-proxy]]).

## 트레이드오프 / 한계

- **`StackOverflowError`**: 무한 재귀 → 호출마다 스택 프레임이 쌓임 → 스택 고갈.
  스택 크기는 힙보다 훨씬 작고 `-Xss` 로 조정한다.
- **힙 크기**는 `-Xms`(초기) / `-Xmx`(최대) 로 지정한다. 크게 잡으면 GC 주기가 길어지는 대신
  한 번의 정지 시간이 길어지는 교환이 있다.
- 참조가 스택에 있다는 점 때문에 **참조를 넘겨도 객체는 복사되지 않는다.**
  자바의 인자 전달은 언제나 값 전달(pass-by-value)이지만, 그 "값"이 참조라서 호출된 메서드가
  같은 힙 객체를 수정할 수 있다.

> [!WARNING]
> **오답 코너**
> - **"힙이 스레드마다 있다"** — 정반대다. **스택이 스레드마다 1개씩, 힙은 공유 1개.**
>   힙이 스레드별로 분리되면 객체 공유 자체가 불가능해진다.
> - **"참조 변수 안에 객체가 들어있다"** — 들어있는 것은 **힙 객체의 주소**다.
> - **"메서드가 끝나면 객체도 사라진다"** — 스택 프레임은 사라지지만 힙 객체는 남는다.
>   참조가 하나도 남지 않았을 때 GC 대상이 될 뿐이다.
> - **"지역변수도 동기화해야 안전하다"** — 다른 스레드가 접근할 경로가 없어 이미 안전하다.

### 스택 고갈은 `Exception` 이 아니라 `Error` 다

```
Throwable
├── Exception   ← 애플리케이션이 대응할 수 있는 상황 (복구 시도 가능)
└── Error       ← JVM 수준에서 무너진 상황 (복구 대상이 아님)
    ├── StackOverflowError
    └── OutOfMemoryError
```

이 구분은 실무에서 치명적이다. `Error` 는 `Exception` 의 하위가 아니므로

```java
@ExceptionHandler(Exception.class)   // StackOverflowError 는 여기 걸리지 않는다
```

전역 예외 핸들러를 그대로 통과해 WAS 까지 올라간다. 로그에는 동일한 스택 프레임이 수천 줄 반복되고
사용자에겐 정제되지 않은 500 이 나간다. 게다가 스택이 이미 무너진 상태라 `catch` 안에서 뭘 하려 해도 다시 터질 수 있다.
**고칠 대상은 예외 처리가 아니라 스택을 고갈시킨 코드 자체다.**

`Exception` 쪽은 다시 **`RuntimeException` 을 상속했는지**로 Checked/Unchecked 가 갈린다.
`Error` 는 Checked 강제(catch or declare)를 받지 않고, `@Transactional` 에서는 `RuntimeException` 과 함께 **기본 롤백 대상**이다
([[checked-vs-unchecked-exception]], [[transactional-rollback-rules]]).

무한 루프와 무한 재귀는 다르다. `while(true)` 는 프레임을 쌓지 않아 영원히 돌지만,
재귀는 호출마다 프레임을 쌓으므로 몇 초 안에 죽는다.
엔티티의 양방향 연관을 `equals` 로 비교할 때가 대표적인 사례다 ([[jpa-entity-equality]]).

### 지역변수의 스레드 안전이 만드는 실무 판단

지역변수가 구조적으로 도달 불가능하다는 사실은 "동기화가 필요 없다"에서 끝나지 않는다.
**메서드 안에서 만들어 밖으로 새어나가지 않는 객체에 락을 거는 것은 순수한 낭비**라는 판단으로 이어진다.
`StringBuffer` 대신 `StringBuilder` 를 쓰는 근거가 정확히 이것이다 ([[string-concatenation-cost]]).

## 복습 체크

- [ ] `int count` 와 `new Pizza()` 와 `p` 가 각각 어디에 놓이는지 말할 수 있는가?
- [ ] 스레드가 3개일 때 스택과 힙은 각각 몇 개인가?
- [ ] 지역변수가 스레드 안전한 이유를 "구조적 도달 불가"로 설명할 수 있는가?
- [ ] 힙이 공유되지 않으면 무엇이 불가능해지는가?
- [ ] `StackOverflowError` 와 `OutOfMemoryError` 가 각각 어느 영역의 고갈인가?
- [ ] 싱글턴 빈에 가변 필드를 두면 왜 위험한가?
- [ ] `StackOverflowError` 가 `Exception` 이 아니라 `Error` 인 것이 왜 실무에서 문제가 되는가?
- [ ] 무한 루프와 무한 재귀는 무엇이 다른가?
- [ ] 지역변수로 만든 객체에 동기화를 거는 것이 왜 낭비인가?

## 관련

[[garbage-collection-reachability]] · [[jvm-execution-pipeline]] · [[interpreter-and-jit]] · [[db-pagination]] · [[jpa-entity-equality]] · [[string-concatenation-cost]] · [[checked-vs-unchecked-exception]] · [[transactional-rollback-rules]] · [[race-condition-lost-update]] · [[static-util-vs-spring-bean]]
