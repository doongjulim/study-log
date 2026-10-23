---
type: refined
slug: switch-fall-through
tags: [switch, switch문, switch-statement, fall-through, 폴스루, 흘러떨어짐, case라벨, case-label, 라벨, label, 점프, jump, break, default, 콜론형식, colon-case, 조건검사한번, case묶기, case-grouping, 중복case, duplicate-case-label, break누락, missing-break, if-else사슬오해, 면접답변]
topic: "옛 switch(콜론 형식)는 값에 맞는 case 라벨로 한 번 점프한 뒤 아래 라벨을 검사 없이 지나쳐 break 나 블록 끝까지 흘러 떨어진다"
summary: "switch 는 if-else 처럼 case 마다 조건을 검사하지 않는다. 진입할 때 한 번 값에 맞는 case 라벨로 점프하고, 그 아래의 case·default 는 표지판일 뿐이라 검사 없이 지나치며 break 나 블록 끝에서만 멈춘다(fall-through). 그래서 break 없는 코드에서 day = 2 면 화·수·기타가 모두 출력된다. 한 값은 case 하나에만 맞으므로 fall-through 의 쓸모는 여러 값이 한 처리를 공유하는 case 묶기이고, break 를 하나 빠뜨리면 다음 case 까지 실행되는 함정이 된다."
contributors: [dongju]
source_refs:
  - https://docs.oracle.com/javase/specs/jls/se17/html/jls-14.html#jls-14.11.3
  - https://docs.oracle.com/javase/tutorial/java/nutsandbolts/switch.html
updated: 2026-10-23
---

# switch fall-through

## 개념 정의 — 라벨로 점프하고 아래로 흐른다

```java
int day = 2;

switch (day) {
    case 1:
        System.out.println("월");
    case 2:
        System.out.println("화");
    case 3:
        System.out.println("수");
    default:
        System.out.println("기타");
}
// 출력: 화 / 수 / 기타
```

```
switch (day)  →  day == 2  →  "case 2:" 라벨로 한 번에 점프
                                 ↓
                 System.out.println("화");   실행
                 case 3:                     ← 라벨일 뿐. 검사하지 않고 지나침
                 System.out.println("수");   실행
                 default:                    ← 라벨일 뿐. 검사하지 않고 지나침
                 System.out.println("기타"); 실행
                 }                           ← 블록 끝에서 멈춤
```

- **조건 검사는 진입할 때 딱 한 번.** 이후의 `case`·`default` 는 아무 의미 없는 표지판이다.
- 실행은 `break` 나 블록 끝까지 **아래로 흘러 떨어진다** → fall-through.
- `break` 가 거의 모든 switch 예제에 붙어 있는 이유가 이것이다. switch 가 맞는 case 만 실행한다면 `break` 는 필요 없었을 것이다.

## 작동 원리 — 쓸모는 case 묶기

```java
switch (day) {
    case 6:
    case 7:
        System.out.println("주말");
        break;
    default:
        System.out.println("평일");
}
```

| 입력 | 추적 | 출력 |
|---|---|---|
| `day = 6` | `case 6:` 로 점프 → `case 7:` 라벨은 그냥 지나침 → "주말" → `break` | 주말 |
| `day = 3` | 맞는 case 없음 → `default:` 로 점프 | 평일 |
| `day = 6`, `break` 삭제 | "주말" → `default:` 라벨 지나침 → "평일" | 주말, 평일 |

한 값은 case **하나**에만 맞는다. 같은 값의 case 를 두 번 쓰면 `duplicate case label` 컴파일 에러다. 그래서 fall-through 의 쓸모는 "한 값이 여러 조건을 충족"이 아니라 **"여러 값이 한 처리를 공유"** 다.

## 트레이드오프 / 한계

- `break` 하나 빠뜨리면 토요일이 **주말이면서 평일**이 된다. 컴파일러는 경고하지 않는다.
- 자바 14 의 **화살표(`->`) 형식**은 fall-through 가 없어 이 함정을 없앤다 ([[switch-expression]]). 콜론 형식은 여전히 fall-through 한다.

> [!WARNING]
> **오답 코너**
> - **"switch 는 모든 case 를 검사해서 맞는 곳만 실행한다"** — if-else 사슬 모델이다. 그렇다면 `break` 가 필요 없다. 실제는 **라벨로 점프 → 아래로 흐름 → break**.
> - **"내려가면서 조건에 맞는 case 에 들른다"** — 진입 뒤에는 **어떤 라벨도 검사하지 않는다**. `day = 2` 여도 "수"가 찍힌다.
> - **"fall-through 는 조건을 여러 개 충족할 때 쓴다"** — 한 값은 case 하나에만 맞는다. **여러 값 → 한 처리**를 묶을 때 쓴다.

## 면접 답변

> "switch 는 값에 맞는 case 라벨로 점프한 뒤, 그 아래 case 들은 다시 검사하지 않고 break 를 만날 때까지 아래로 계속 실행합니다. 이걸 fall-through 라고 합니다. 여러 값이 같은 처리를 공유할 때 case 를 묶는 데 쓸 수 있지만, break 를 빠뜨리면 의도하지 않은 case 까지 실행되는 버그가 됩니다."

## 복습 체크

- [ ] break 없는 switch 에서 `day = 2` 의 출력과 이유는?
- [ ] "case 라벨은 표지판"이라는 말을 실행 흐름으로 설명할 수 있는가?
- [ ] 주말/평일 판별을 fall-through 로 쓰고, `day = 6`·`day = 3`·break 삭제 시를 추적할 수 있는가?
- [ ] fall-through 의 쓸모와 함정은?

## 관련

[[switch-expression]] · [[enum-constant-specific-polymorphism]] · [[java17-lts-features]]
