---
type: refined
slug: java-pass-by-value
tags: [call-by-value, pass-by-value, 값에의한호출, 값전달, call-by-reference, pass-by-reference, 참조에의한호출, 매개변수, parameter, 인자, argument, 참조복사, reference-copy, 대입, assignment, 재대입, reassignment, 앨리어싱, aliasing, 공유객체, 스택프레임, stack-frame, 지역변수, swap, C++참조, ref, 숨은대입문, 면접답변]
topic: "자바는 항상 Call by Value이고 참조형은 참조값을 복사한다 — 메서드 호출은 숨은 대입문 한 줄이다"
summary: "대입은 변수 칸의 내용을 복사하는 것이라 int b = a 는 값을, Member m2 = m1 은 참조를 복사해 같은 객체를 공유(앨리어싱)한다. 메서드 호출은 매개변수라는 새 지역변수에 인자의 내용을 복사하는 숨은 대입문(Member p = m)과 같아서, 복사된 참조로 객체 필드를 바꾸면 호출한 쪽에도 보이지만 매개변수에 새 객체를 재대입해도 호출한 쪽 변수는 바뀌지 않는다."
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-15.html#jls-15.12.4.5
updated: 2026-10-22
---

# Call by Value — 자바의 인자 전달

## 개념 정의 — 대입은 칸의 내용을 복사한다

```java
int a = 3;
int b = a;          // 값 3 을 복사 → 별개의 칸
b = 10;
System.out.println(a);        // 3

Member m1 = new Member();
m1.age = 20;
Member m2 = m1;     // 참조를 복사 → 객체는 하나, 가리키는 변수가 둘
m2.age = 30;
System.out.println(m1.age);   // 30
```

- 변수에 무엇이 들어 있는지(값 자체 vs 참조)가 복사되는 대상을 정한다 ([[primitive-vs-reference-type]]).
- 두 변수가 같은 객체를 가리키는 상태를 **앨리어싱(aliasing)** 이라 한다. 공유 리스트나 값 타입 공유 버그의 뿌리다 ([[shallow-immutability-defensive-copy]], [[jpa-embedded-value-type]]).

## 작동 원리 — 메서드 호출 = 숨은 대입문

```java
void changeAge(Member p) {
    p.age = 99;
}

void replace(Member p) {
    p = new Member();
    p.age = 1;
}

Member m = new Member();
m.age = 20;

changeAge(m);
System.out.println(m.age);   // (A) 99

replace(m);
System.out.println(m.age);   // (B) 99
```

호출하면 매개변수 `p` 가 **새 지역변수**로 스택 프레임에 만들어지고, 인자의 내용이 복사된다. 그래서 `replace(m)` 은 사실상 이렇게 실행된다.

```java
Member p = m;            // ① 호출: m 안의 참조를 p 로 복사 → 같은 객체
p = new Member();        // ② p 의 칸에만 새 객체의 참조를 재대입
p.age = 1;               // ③ 새 객체의 age = 1
System.out.println(m.age);   // m 은 처음 객체(99)를 계속 가리킴 → 99
```

| | `changeAge(m)` | `replace(m)` |
|---|---|---|
| 하는 일 | 복사된 참조로 **같은 객체의 필드**를 바꿈 | 복사된 **지역변수 `p` 의 칸**에 새 참조를 재대입 |
| 호출한 쪽 `m` 에 보이나? | ⭕ 같은 객체니까 | ❌ `m` 의 칸은 그대로 |

### 판정 기준 — Call by Reference 였다면

- 진짜 Call by Reference(C++ `Member& p`, C# `ref`)라면 `p` 는 `m` 의 **별명** 자체가 된다. `p = new Member()` 가 호출한 쪽 `m` 까지 바꾸므로 (B) 는 1 이 된다.
- 자바에서 (B) 는 99 다. **자바는 항상 Call by Value 이고, 참조형은 참조값을 복사한다.**
- `int b = a; b = 10;` 다음에 `a` 가 그대로인 것과 같은 원리다. 칸에 새 값을 대입하면 그 칸만 바뀐다. 참조형이라고 다르지 않다.

## 트레이드오프 / 한계

- **swap 메서드를 만들 수 없다.** `swap(Member x, Member y)` 안에서 `x`, `y` 를 바꿔도 호출한 쪽 변수는 그대로다. 바꾼 결과를 반환하거나, 배열·홀더 객체에 담아 넘겨야 한다.
- **"객체 상태 변경이 보인다"는 성질은 양날의 검이다.** 넘겨받은 컬렉션이나 가변 객체를 메서드가 수정하면 호출한 쪽도 바뀐다. 의도하지 않은 공유를 막으려면 방어적 복사나 불변 객체를 쓴다 ([[shallow-immutability-defensive-copy]]).

> [!WARNING]
> **오답 코너**
> - **"메서드 안에서 필드를 바꾸면 호출한 쪽도 바뀌니 Call by Reference"** — 공유되는 것은 **변수가 아니라 객체**다. 매개변수에 새 객체를 대입해도 호출한 쪽은 바뀌지 않는다 → **Call by Value**.
> - **"`replace(m)` 후 `m.age` 는 1"** — **99** 다. 바뀌는 것은 지역변수 `p` 의 칸뿐이다.
> - **"`p = new Member()` 는 선언을 새로 한 것"** — **재대입**이다. 선언이면 타입이 붙고, 이미 있는 매개변수 이름이라 컴파일 에러가 난다.
> - **"참조가 스택에 있어서 객체가 복사되지 않는다"** — 원인은 위치가 아니라 **변수에 담긴 것이 참조**라는 점이다.

## 면접 답변

> "자바는 항상 Call by Value입니다. 참조형도 참조값이 복사되어 넘어가기 때문에, 메서드 안에서 그 참조로 객체의 상태를 바꾸면 호출한 쪽에도 보이지만, 매개변수에 새 객체를 대입해도 호출한 쪽 변수는 바뀌지 않습니다."

## 복습 체크

- [ ] `int b = a` 와 `Member m2 = m1` 에서 각각 무엇이 복사되는가?
- [ ] 메서드 호출을 대입문 한 줄로 바꿔 쓸 수 있는가?
- [ ] `changeAge(m)` 과 `replace(m)` 의 결과가 다른 이유는?
- [ ] Call by Reference 였다면 `replace(m)` 후 `m.age` 는 몇이어야 하는가?
- [ ] 자바로 swap 메서드를 만들 수 없는 이유는?

## 관련

[[primitive-vs-reference-type]] · [[jvm-stack-and-heap]] · [[shallow-immutability-defensive-copy]] · [[jpa-embedded-value-type]] · [[reference-vs-value-equality]]
