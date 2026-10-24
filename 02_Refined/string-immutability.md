---
type: refined
slug: string-immutability
tags: [String, 문자열, 불변, immutable, 불변객체, 문자열상수풀, String-Pool, 상수풀, intern, 상수폴딩, constant-folding, 해시캐싱, hash-cache, 스레드안전, thread-safe, 리터럴, literal, new-String, CompactStrings, 값객체]
topic: String 이 불변인 이유와, 그 불변성 하나가 낳는 세 가지 이득
summary: 불변성은 해시 캐싱·스레드 안전·상수 풀 공유라는 세 이득을 나란히 낳는다. 리터럴은 풀에서 공유되고 컴파일 타임 상수는 폴딩되지만 런타임 연산 결과는 힙의 새 객체이므로, 문자열 비교는 언제나 equals 여야 한다.
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/String.html
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-3.html#jls-3.10.5
updated: 2026-10-24
---

# String 의 불변성

## 개념 정의

```
불변(immutable) ─┬─→ 해시 캐싱      한 번 계산한 해시가 영원히 유효
                 ├─→ 스레드 안전     아무도 못 바꾸니 경쟁 상태가 성립 불가
                 └─→ 상수 풀 공유    같은 리터럴을 여러 곳이 안심하고 공유
```

**원인은 불변성 하나, 결과가 셋이다.** 셋을 서로의 원인으로 엮으면 안 된다
(해시 캐싱 "때문에" 스레드 안전해지는 것이 아니다).

```java
String s = "a";
s = s + "b";
s = s + "c";
```

`"a"` 객체는 끝까지 변하지 않았다. **변한 것은 객체가 아니라 `s` 가 가리키는 대상**이다.
이 3줄이 만드는 `String` 객체는 **5개** — 리터럴 `"a"`, `"b"`, `"c"`(상수 풀)와
런타임 연산 결과 `"ab"`, `"abc"`(힙). 저장 영역도 다르다.

## 작동 원리

### 상수 풀과 컴파일 타임 상수 폴딩

```java
String a = "abc";
String b = "abc";
String c = "ab" + "c";        // 양쪽 다 리터럴
String e = "ab";
String f = e + "c";           // 변수가 끼면 런타임 연산
String d = new String("abc");
```

```
a == b : true
a == c : true    ← 컴파일 타임 상수 폴딩. javac 이 "abc" 로 합쳐 풀에 넣는다
a == f : false   ← 런타임 연산 결과는 힙의 새 객체
a == d : false
a == f.intern() : true
```

`javap -c` 로 보면 폴딩은 바이트코드에서 눈으로 확인된다. `final` + 리터럴 초기화 변수(**상수 변수**)도 컴파일러가 코드만 읽고 값을 확신하므로 똑같이 폴딩된다.

```java
String s1 = "javaspring";
String s2 = "java" + "spring";      // s1 == s2  true
String part = "spring";
String s3 = "java" + part;          // s1 == s3  false
final String fpart = "spring";
String s4 = "java" + fpart;         // s1 == s4  true   ← 상수 변수라 폴딩
String s5 = s3.intern();            // s1 == s5  true,  s3 == s5  false
```

```
 0: ldc   "javaspring"      ← s1
 3: ldc   "javaspring"      ← s2   (덧셈이 사라짐 = 폴딩)
10: invokedynamic makeConcatWithConstants   ← s3   (실행 중 연결)
17: ldc   "javaspring"      ← s4   (덧셈이 사라짐 = 폴딩)
```

| 코드 | 컴파일러가 값을 확신? | 결과 |
|---|---|---|
| `"java" + "spring"` | ⭕ 리터럴끼리 | 컴파일 때 폴딩 → 풀 |
| `"java" + part` | ❌ 일반 변수 | 실행 중 연결 → 힙의 새 객체 |
| `"java" + fpart` | ⭕ 상수 변수(`final` + 리터럴 초기화) | 폴딩 → 풀 |

`new String("abc")` 는 객체를 **최대 2개** 만든다. 괄호 안 리터럴(풀에 없을 때만)과 힙의 새 객체(항상)다. 풀에 이미 있으면 1개다.

**구조가 래퍼 캐시와 동일하다** ([[wrapper-cache-autoboxing]]).
미리 만들어둔 것을 재사용하면 `==` 가 우연히 `true`, 런타임에 새로 만들면 `false`.
그래서 문자열 비교는 언제나 `equals` 여야 한다 ([[reference-vs-value-equality]]).

### `private int hash` — 런타임 지연 캐싱

```java
public int hashCode() {
    int h = hash;
    if (h == 0 && !hashIsZero) { h = ...계산...; hash = h; }
    return h;
}
```

컴파일러가 아니라 **런타임에 처음 호출될 때 계산해 필드에 저장**한다.
긴 문자열을 `HashMap` key 로 100번 조회해도 **실제 계산은 1번**, 나머지는 필드 값을 반환한다.

이 최적화가 가능한 이유가 곧 불변성이다. 내용이 절대 안 변하니 한 번 계산한 해시가 영원히 유효하다.
`String` 이 가변이었다면 [[equals-hashcode-contract]] 의 유령 원소 사고가 `HashMap` 의 모든 key 에서 일어났을 것이다.

### 공유가 안전한 이유는 `final` 이 아니라 불변성

String 에는 `append`·`setCharAt` 처럼 **자기 내용을 바꾸는 메서드가 없다.** `toUpperCase()` 같은 메서드는 원본을 두고 **새 String 객체**를 돌려준다. 그래서 상수 풀에서 여러 변수가 한 객체를 공유해도 누구도 다른 쪽을 오염시킬 수 없다.

`final` 과 불변은 다르다. `final StringBuilder sb` 도 `sb.append("!")` 는 된다 — `final` 은 참조만 고정한다. `String` **클래스에 붙은 `final`(상속 금지)은 보조 장치**다. 내용을 바꾸는 메서드를 추가한 하위 클래스가 생기면 풀에서 공유하던 문자열이 모두 바뀔 수 있으므로 그 길을 막는다.

### 스레드 안전

여러 스레드가 같은 객체를 참조하는데 **아무도 상태를 바꿀 수 없다면** 경쟁 상태가 원천적으로 성립하지 않는다.
락도, `synchronized` 도, `volatile` 도 필요 없다. **불변 객체는 그 자체로 thread-safe 다.**

## 트레이드오프 / 한계

- 불변의 대가는 **연결할 때마다 새 객체**라는 점이다. 반복 누적에서 O(n²) 가 되는 이유가 이것이고,
  그래서 가변 버퍼를 쓰는 `StringBuilder` 가 필요하다 ([[string-concatenation-cost]]).
- `String` 은 **깊은 불변**이지만, 모든 필드가 `final` 인 클래스가 곧 불변은 아니다. `final` 은 참조만 고정하므로
  가변 컬렉션을 담으면 내용이 바뀐다 → 방어적 복사가 필요하다 ([[shallow-immutability-defensive-copy]]).
- `intern()` 은 **내용이 같은 풀의 대표 객체를 돌려준다.** 풀에 없으면 등록한 뒤 그 객체를 돌려준다. 이미 있으면 호출한 힙 객체는 그대로 남는다(`s3 == s3.intern()` 이 false 일 수 있다). 호출 비용이 있고, 대량 사용 시 풀이 커진다.

  > 정정 (2026-10-24): 이전 판은 "`intern()` 은 힙의 문자열을 풀에 넣어 공유시킨다"고 적었다. 풀에 같은 내용이 이미 있으면 힙 객체를 풀로 옮기지 않고 **기존 풀 객체를 반환**할 뿐이다. 근거: 위 예제에서 `s1 == s3.intern()` 은 true, `s3 == s3.intern()` 은 false (실행 확인). [String.intern Javadoc](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/String.html#intern())
- 비밀번호 같은 민감값을 `String` 으로 들고 있으면 **명시적으로 지울 수 없다.**
  가변인 `char[]` 를 쓰고 사용 후 덮어쓰라는 권고가 여기서 나온다.

> [!WARNING]
> **오답 코너**
> - **"`s = s + "b"` 는 `"a"` 객체를 바꾼다"** — 객체는 그대로고 **변수가 가리키는 대상**이 바뀐다.
> - **"`"abc" == "ab" + "c"` 는 `false`"** — **`true`.** 양쪽이 컴파일 타임 상수라 `"abc"` 로 폴딩되어 풀에 들어간다.
>   변수가 하나라도 끼면(`e + "c"`) 런타임 연산이라 `false` 다.
> - **"리터럴 공유가 안전한 건 String 이 `final` 이라서"** — **불변성** 때문이다. 클래스 `final` 은 상속으로 불변성이 깨지는 걸 막는 보조 장치이고, `final StringBuilder` 는 여전히 바뀐다.
> - **"`new String("abc")` 는 객체 1개"** — **최대 2개**(풀 리터럴 + 힙). 풀에 이미 있으면 1개.
> - **"`"java" + "spring"` 은 변수에 담기는 시점에 풀에 올라간다"** — **컴파일할 때** 폴딩된다. 그 규칙이면 `"java" + part` 도 true 여야 한다.
> - **"`final` 이면 컴파일러가 메서드를 호출해 생성하는 시점에 확신한다"** — 컴파일러는 실행하지 않는다. `final` + 리터럴 초기화 = 상수 변수라 코드만 읽고 확신한다.
> - **"`s3.intern()` 은 s3 를 풀로 옮긴다"** — 풀의 대표 객체를 **돌려줄 뿐**이다. `s1 == s3.intern()` true, `s3 == s3.intern()` false.
> - **"`hash` 캐싱은 컴파일러가 해준다"** — **런타임 지연 캐싱**이다.
> - **"불변 → 해시캐싱 → 스레드안전"** — 직렬 인과가 아니다.
>   불변이 원인이고 **세 이득이 나란히** 따라온다.

## 복습 체크

- [ ] `String` 이 불변이라는 말의 정확한 의미를 변수/객체를 구분해 설명할 수 있는가?
- [ ] `String s="a"; s=s+"b"; s=s+"c";` 로 만들어지는 객체는 몇 개이며 어디에 저장되는가?
- [ ] `a == c` 가 `true` 이고 `a == f` 가 `false` 인 이유는?
- [ ] `private int hash` 는 무엇이며 왜 불변이어야 가능한가?
- [ ] 불변성이 낳는 세 이득을 각각 말할 수 있는가?
- [ ] 비밀번호를 `String` 대신 `char[]` 로 다루라는 권고의 근거는?
- [ ] `new String("abc")` 가 만드는 객체는 최대 몇 개인가?
- [ ] 상수 풀 공유를 가능하게 하는 성질과, 클래스 `final` 의 역할을 구분할 수 있는가?
- [ ] `"java" + part` 와 `"java" + fpart`(final) 의 결과가 다른 이유를 "컴파일러가 값을 확신하는가"로 설명할 수 있는가?
- [ ] `intern()` 이 돌려주는 것은 무엇이고, `s3 == s3.intern()` 은 왜 false 일 수 있는가?

## 관련

[[string-concatenation-cost]] · [[reference-vs-value-equality]] · [[numeric-promotion]] · [[wrapper-cache-autoboxing]] · [[equals-hashcode-contract]] · [[jvm-stack-and-heap]] · [[shallow-immutability-defensive-copy]]
