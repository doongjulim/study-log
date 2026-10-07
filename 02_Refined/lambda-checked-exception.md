---
type: refined
slug: lambda-checked-exception
tags: [람다, lambda, 람다예외, lambda-exception, checked-exception, 체크예외, 함수형인터페이스, functional-interface, Predicate, Function, Consumer, Supplier, throws, catch-or-declare, IOException, UncheckedIOException, 예외감싸기, exception-wrapping, Stream예외, stream-exception, Files.readString, 메서드추출, method-extraction, ThrowingFunction]
topic: 람다 안에서 checked 예외를 그대로 던질 수 없는 이유 — 함수형 인터페이스 메서드 시그니처에 throws 가 없다
summary: 람다는 Predicate.test, Function.apply 같은 함수형 인터페이스 메서드의 구현이고, 구현 메서드는 인터페이스가 선언하지 않은 checked 예외를 던질 수 없다. 표준 함수형 인터페이스에는 throws 가 없으므로 catch-or-declare 중 catch 만 남아, 람다 안에서 try-catch 로 잡아 UncheckedIOException 등으로 감싸야 한다. 이 소음 때문에 checked 예외를 던지는 I/O 중심 루프는 Stream 보다 for 문이 읽기 쉽다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/function/Predicate.html
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/io/UncheckedIOException.html
updated: 2026-10-07
---

# 람다와 checked 예외

## 개념 정의

일반 메서드는 `throws IOException` 만 붙이면 예외를 호출자에게 넘길 수 있다 ([[checked-vs-unchecked-exception]] 의 catch-or-declare). 그런데 람다 안에서는 안 된다.

```java
paths.stream()
     .filter(path -> Files.readString(Path.of(path)).contains("ERROR"))   // ❌ 컴파일 에러
     .findFirst();
// error: unreported exception IOException; must be caught or declared to be thrown
```

## 작동 원리

`filter` 가 받는 람다는 `Predicate` 의 구현체다:

```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);    // ← throws 선언이 없다
}
```

1. 람다 본문 = `test()` 메서드의 **구현**.
2. 오버라이드 규칙: 구현 메서드는 인터페이스 메서드가 선언한 것보다 **더 많은 checked 예외를 던질 수 없다.**
3. `test()` 는 아무것도 선언하지 않았다 → **declare 불가** → catch-or-declare 중 **catch 만** 남는다.

`Function.apply`, `Consumer.accept`, `Supplier.get` 도 모두 같다.

```java
paths.stream()
     .filter(path -> {
         try {
             return Files.readString(Path.of(path)).contains("ERROR");
         } catch (IOException e) {
             throw new UncheckedIOException(e);   // unchecked 로 감싸서 탈출
         }
     })
     .findFirst()
     .ifPresent(this::alert);
```

원래 for 문보다 길고, 선언형의 장점이 try-catch 소음에 묻힌다.

```java
for (String path : paths) {
    String content = Files.readString(Path.of(path));   // 메서드에 throws IOException 만 붙이면 끝
    if (content.contains("ERROR")) { alert(path); break; }
}
```

## 트레이드오프 / 한계 — 선택지

| 방법 | 장점 | 단점 |
|---|---|---|
| **for 문으로 둔다** | 가장 단순, `throws` 그대로 전파 | 선언형 아님 |
| 람다 안 try-catch → `UncheckedIOException` | Stream 유지 | 소음, 호출부에서 원래 타입으로 catch 불가 |
| **메서드 추출** (`this::containsError` 안에서 감싸기) | 파이프라인은 깔끔 | 감싸기 로직이 숨을 뿐 사라지진 않음 |
| 커스텀 `ThrowingFunction` 인터페이스 | throws 선언 가능 | 표준 Stream API 는 받지 않음 → 결국 어댑터 필요 |

- 감싼 예외(`UncheckedIOException`)는 RuntimeException 이라 `@Transactional` 기본 롤백 대상이 되는 등 **의미가 바뀐다** ([[transactional-rollback-rules]]).
- 판단 기준: **checked 예외를 던지는 I/O 가 루프의 중심이면 for 문**이 낫다 ([[stream-vs-for-loop]]).

> [!WARNING]
> **오답 코너**
> - **"람다에서 checked 예외를 못 던진다"(이유 없이 결론만)** — 면접에선 이유까지 말해야 한다. **함수형 인터페이스 메서드(예 `Predicate.test`)에 throws 가 없어서** 구현인 람다도 선언할 수 없다.
> - **"람다를 감싼 메서드에 throws 를 붙이면 된다"** — 람다 본문은 바깥 메서드가 아니라 `test()` 의 구현이다. 바깥 메서드의 throws 와 무관하다.

## 복습 체크

- [ ] 람다 안에서 `IOException` 이 컴파일 에러가 나는 이유를 오버라이드 규칙으로 설명할 수 있는가?
- [ ] catch-or-declare 중 왜 catch 만 남는가?
- [ ] `UncheckedIOException` 으로 감쌀 때 잃는 것은?
- [ ] checked 예외 I/O 루프에서 for 문을 택하는 근거를 말할 수 있는가?

## 관련

[[checked-vs-unchecked-exception]] · [[stream-vs-for-loop]] · [[transactional-rollback-rules]] · [[optional-orelse-vs-orelseget]]
