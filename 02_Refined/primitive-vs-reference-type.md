---
type: refined
slug: primitive-vs-reference-type
tags: [기본형, primitive-type, 원시타입, 참조형, reference-type, 참조, reference, 변수의내용, 값자체, 기본형8개, byte, short, int, long, float, double, char, boolean, 기본값, default-value, 지역변수, local-variable, 필드, field, 초기화안된변수, definite-assignment, 메모리위치, memory-layout, 스택, stack, 힙, heap, 그릇을따라간다, 객체헤더, object-header, 간접참조, indirection, 캐시미스, GC, 문자열상수풀, string-pool, 래퍼, wrapper, 프록시오용, 면접답변]
topic: "기본형과 참조형은 위치가 아니라 변수 안에 값 자체가 있느냐 참조가 있느냐로 갈리고, 메모리 위치는 그 변수를 담은 그릇을 따라간다"
summary: "기본형 8개는 변수 칸에 값 자체가, 참조형은 힙 객체를 가리키는 참조가 들어 있다. '기본형은 스택'은 틀린 일반화로, 지역변수면 스택이고 객체 필드나 배열 칸이면 힙이다. 기본값(0/null)은 필드만 받고 지역변수는 초기화 전에 읽으면 컴파일 에러다. 기본형이 존재하는 이유는 객체 헤더·간접 참조·GC 비용 없이 숫자를 다루기 위해서다."
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-4.html
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-16.html
updated: 2026-10-22
---

# 기본형 vs 참조형

## 개념 정의 — 차이는 "변수 안에 무엇이 들어 있나"

| | 변수 안에 들어 있는 것 | null 가능 | 기본값 (필드만) |
|---|---|---|---|
| 기본형 8개: `byte` `short` `int` `long` `float` `double` `char` `boolean` | **값 자체** | ❌ | 0 / 0L / 0.0 / false / `'\u0000'` |
| 참조형 (클래스, 배열, 인터페이스, enum …) | 힙 객체를 가리키는 **참조** | ⭕ | null |

- `int count = 3` 이면 `count` 라는 4바이트 칸에 **3의 비트 패턴이 직접** 들어 있다. 어딘가에 미리 만들어 둔 3을 가리키는 것이 아니다.
- 미리 만든 객체를 재사용하는 것은 `Integer.valueOf` 캐시(-128~127), 즉 **참조형 `Integer`** 의 이야기다 ([[wrapper-cache-autoboxing]]).

### 기본값은 필드만 받는다

```java
void test() {
    int x;
    System.out.println(x);   // 컴파일 에러: variable x might not have been initialized
}
```

- 지역변수는 기본값이 없다. 컴파일러가 "값을 넣기 전에 읽는 코드"를 막는다(definite assignment).
- 기본값을 받는 것은 **static 필드**(클래스 로딩 준비 단계: [[class-loading-lifecycle]])와 **인스턴스 필드**(`new` 시점)뿐이다.

## 작동 원리 — 메모리 위치는 그릇을 따라간다

```java
class Member {
    int age;
    String name;
}

void join() {
    int count = 3;
    Member m = new Member();
    m.age = 20;
    m.name = "DJ";
}
```

| 대상 | 위치 | 들어 있는 것 |
|---|---|---|
| `count` | 스택 (지역변수) | 값 3 |
| `m` | 스택 (지역변수) | `Member` 객체를 가리키는 참조 |
| `Member` 객체 | 힙 | `age` 칸, `name` 칸 |
| `age` | **힙** (객체 안의 필드) | 값 20 |
| `name` | 힙 (객체 안의 필드) | `"DJ"` String 객체를 가리키는 참조 |
| `"DJ"` | 힙 (문자열 상수 풀, Java 7부터 힙 안) | 글자 데이터 |

> **"기본형은 스택에 있다"는 틀린 일반화다.** 기본형이든 참조든 **담은 그릇이 있는 곳**에 함께 있다. 지역변수면 스택, 객체 필드나 배열 칸이면 힙이다.
> 기본형과 참조형을 가르는 기준은 **위치가 아니라 내용**(값 자체냐 참조냐)이다.

이 차이가 대입과 메서드 호출의 동작을 결정한다. 대입은 칸의 내용을 복사하므로 기본형은 값이, 참조형은 참조가 복사된다 ([[java-pass-by-value]]). `==` 가 무엇을 비교하는지도 같은 원리다 ([[reference-vs-value-equality]]).

## 트레이드오프 — 기본형이 존재하는 이유

```java
int[]     a = new int[1_000_000];
Integer[] b = new Integer[1_000_000];   // 각 칸에 Integer 객체를 채운 상태
```

| | `int[1_000_000]` | `Integer[1_000_000]` (객체 채움) |
|---|---|---|
| 칸에 들어 있는 것 | 숫자 자체 (4B) | 참조 (압축 참조면 4B) |
| 추가 비용 | 없음 | 객체 100만 개 × 약 16B (헤더 12B + 값 4B) |
| 대략 총량 | 약 4MB | 약 20MB (64비트 JVM, 압축 참조 기준) |
| 숫자 하나 읽기 | 배열 칸 1번 | 배열 칸 → 참조를 따라 객체로 → **2번** (캐시 미스 증가) |
| GC | 배열 1개 | 배열 + **객체 100만 개 추적** |

객체 생성, 메모리, 간접 참조, GC 비용 없이 숫자를 다루기 위해 기본형이 있다.
대가로 **null 을 표현할 수 없다.** "값 없음"이 필요하면 래퍼 타입을 쓰고, 그때는 언박싱 NPE 를 조심해야 한다 ([[unboxing-npe-wrapper-null]]).

> [!WARNING]
> **오답 코너**
> - **"기본형은 `bit` 를 포함해 9개"** — **8개**다. `bit` 는 자바 타입이 아니다.
> - **"변수 안에는 0 이나 null 이 들어 있다"** — 그건 아무것도 넣지 않은 **필드의 기본값**이다. 변수에 담기는 것은 값 자체 또는 참조다.
> - **"`int` 3은 미리 만들어 둔 3을 참조한다"** — 기본형 변수에는 **값 자체**가 들어 있다. `Integer.valueOf` 캐시(참조형)와 혼동한 것이다.
> - **"참조 변수에는 프록시가 들어 있다"** — **참조(reference)** 가 들어 있다. 프록시는 진짜 객체 대신 호출을 가로채는 **대리 객체**(JPA 지연 로딩 [[lazy-loading-proxy]], `@Transactional` [[transactional-proxy-self-invocation]])로 전혀 다른 개념이다.
> - **"참조 변수 `m` 은 힙에 있다"** — 지역변수라 **스택**이다. 힙에 있는 것은 `m` 이 가리키는 객체다.
> - **"기본형은 항상 스택"** — 객체 필드인 `age` 는 **힙**이다. 위치는 그릇을 따라간다.
> - **"초기화하지 않은 지역변수는 0 이 출력된다"** — **컴파일 에러**다.

## 면접 답변

> "기본형은 변수 안에 값 자체가 들어 있고, 참조형은 힙에 있는 객체를 가리키는 참조가 들어 있습니다. 흔히 기본형은 스택에 있다고 하지만, 정확히는 변수를 담은 곳을 따라가서 지역변수면 스택, 객체의 필드면 힙에 있습니다. 기본형이 따로 있는 이유는 숫자 하나하나를 객체로 만들면 객체 헤더와 간접 참조, GC 비용이 들기 때문입니다."

## 복습 체크

- [ ] 기본형 8개를 말할 수 있는가?
- [ ] 기본형 변수와 참조형 변수에 각각 무엇이 들어 있는가?
- [ ] `Member` 예제에서 `count`, `m`, 객체, `age`, `name`, `"DJ"` 의 위치와 내용을 말할 수 있는가?
- [ ] "기본형은 스택"이 틀린 이유를 예로 설명할 수 있는가?
- [ ] 기본값을 받는 변수와 받지 않는 변수는?
- [ ] 모든 숫자를 객체로 만들지 않고 기본형을 둔 이유 세 가지는?

## 관련

[[jvm-stack-and-heap]] · [[java-pass-by-value]] · [[unboxing-npe-wrapper-null]] · [[wrapper-cache-autoboxing]] · [[reference-vs-value-equality]] · [[class-loading-lifecycle]] · [[string-immutability]]
