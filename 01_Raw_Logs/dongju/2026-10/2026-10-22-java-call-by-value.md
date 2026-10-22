---
date: 2026-10-22
type: raw
author: dongju
tags: [call-by-value, pass-by-value, 값에의한호출, 값전달, call-by-reference, pass-by-reference, 참조에의한호출, 매개변수, parameter, 인자, argument, 참조복사, reference-copy, 대입, assignment, 재대입, reassignment, 앨리어싱, aliasing, 공유객체, 스택프레임, stack-frame, 지역변수, swap, C++참조, ref, 면접답변]
topic: 자바는 항상 Call by Value이고 참조형은 참조값을 복사한다 — 메서드 호출은 숨은 대입문 한 줄이다
summary: 대입은 변수 칸의 내용을 복사하는 것이라 int b = a 는 값을, Member m2 = m1 은 참조를 복사해 같은 객체를 공유(앨리어싱)한다. 메서드 호출은 매개변수라는 새 지역변수에 인자의 내용을 복사하는 숨은 대입문(Member p = m)과 같아서, 복사된 참조로 객체 필드를 바꾸면 호출한 쪽에도 보이지만 매개변수에 새 객체를 재대입해도 호출한 쪽 변수는 바뀌지 않는다. 그래서 자바는 항상 Call by Value다.
source: session
distilled: false
---

## 배운 개념

### 1. 대입 = 변수 칸의 내용을 복사

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

- 두 변수가 같은 객체를 가리키는 상태 = **앨리어싱(aliasing)**. 공유 리스트·값 타입 공유 버그의 뿌리 ([[shallow-immutability-defensive-copy]], [[jpa-embedded-value-type]]).

### 2. 메서드 호출 = 숨은 대입문

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

`replace(m)` 은 사실상 이렇게 실행된다.

```java
Member p = m;            // ① 호출: m 안의 참조를 새 지역변수 p 로 복사 → 같은 객체
p = new Member();        // ② p 의 칸에만 새 객체의 참조를 재대입
p.age = 1;               // ③ 새 객체의 age = 1
System.out.println(m.age);   // m 은 처음 객체(99)를 계속 가리킴 → 99
```

| | `changeAge(m)` | `replace(m)` |
|---|---|---|
| 하는 일 | 복사된 참조로 **같은 객체의 필드**를 바꿈 | 복사된 **지역변수 `p` 의 칸**에 새 참조를 재대입 |
| 호출한 쪽 `m` 에 보이나? | ⭕ 같은 객체 | ❌ `m` 의 칸은 그대로 |

### 3. 판정 — Call by Value

- 진짜 Call by Reference(C++ `Member& p`, C# `ref`)라면 `p` 는 `m` 의 **별명** 자체 → `p = new Member()` 가 호출한 쪽 `m` 까지 바꾼다 → (B) 는 1.
- 자바는 (B) = 99 → `m` 은 그대로 → **항상 Call by Value. 참조형은 참조값을 복사한다.**
- `int b = a; b = 10;` 에서 `a` 가 그대로인 것과 **같은 원리** — 칸에 새 값을 대입하면 그 칸만 바뀐다. 참조형이라고 다르지 않다.

### 면접 답변

> "자바는 항상 Call by Value입니다. 참조형도 참조값이 복사되어 넘어가기 때문에, 메서드 안에서 그 참조로 객체의 상태를 바꾸면 호출한 쪽에도 보이지만, 매개변수에 새 객체를 대입해도 호출한 쪽 변수는 바뀌지 않습니다."

## 오답·헷갈린 점

- 👍 대입 예제((1) 3, (2) 30)는 이유까지 정확 — "값 자체를 가져온다 / 같은 곳을 참조한다".
- **전**: `replace(m)` 후 `m.age` = **1** → **후**: **99**. 바뀌는 건 지역변수 `p` 의 칸뿐. 메서드 호출을 `Member p = m;` 대입문으로 펼쳐 보자 바로 정정함.
- **전**: "`p = new Member()` 는 **선언**을 새로 한 것" → **후**: **재대입**. 선언이면 `Member p = ...` 처럼 타입이 붙고, 이미 있는 매개변수 이름이라 컴파일 에러.
- **전**: 그러므로 "자바는 **Call by Reference**로 볼 수 있다" → **후**: **Call by Value**, 참조형은 참조(reference)를 복사. (객체 상태 변경이 보이는 것과 변수 자체가 바뀌는 것을 구분하지 못한 것이 원인)
- **전**: "99 가 나오는 건 `p`" → **후**: 99 는 **`m.age`**. ② 이후 `p` 는 새 객체(age 1)를 가리킨다.

## Q&A

**Q. 자바는 Call by Value 인가 Call by Reference 인가?**
A. 항상 Call by Value. 참조형은 참조값을 복사한다.

**Q. 그런데 메서드 안에서 객체 필드를 바꾸면 왜 호출한 쪽에도 보이나?**
A. 복사된 참조가 **같은 객체**를 가리키기 때문. 변수가 공유된 게 아니라 객체가 공유된 것.

**Q. Call by Reference 였다면 무엇이 달라지나?**
A. 매개변수에 새 객체를 대입하면 호출한 쪽 변수까지 바뀐다(C++ `&`, C# `ref`). 자바에선 안 바뀐다.

## 다음 학습

- swap 메서드를 자바로 못 만드는 이유와 대안(배열·홀더 객체·반환값)
- 방어적 복사가 필요한 이유 — 생성자/게터로 넘긴 컬렉션의 앨리어싱 ([[shallow-immutability-defensive-copy]] 복습)
