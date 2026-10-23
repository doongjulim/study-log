---
date: 2026-10-23
type: raw
author: dongju
tags: [switch, switch문, switch-statement, fall-through, 폴스루, 흘러떨어짐, case라벨, case-label, 라벨, label, 점프, jump, break, default, 콜론형식, colon-case, 조건검사한번, case묶기, case-grouping, 중복case, duplicate-case-label, break누락, missing-break, if-else사슬오해, 면접답변]
topic: 옛 switch(콜론 형식)는 값에 맞는 case 라벨로 한 번 점프한 뒤 아래 라벨을 검사 없이 지나쳐 break 나 블록 끝까지 흘러 떨어진다
summary: switch 는 if-else 처럼 case 마다 조건을 검사하지 않는다. 진입할 때 한 번 값에 맞는 case 라벨로 점프하고, 그 아래의 case·default 는 표지판일 뿐이라 검사 없이 지나치며 break 나 블록 끝에서만 멈춘다(fall-through). 그래서 break 없는 코드에서 day = 2 면 화·수·기타가 모두 출력된다. 한 값은 case 하나에만 맞으므로(중복 case 는 컴파일 에러) fall-through 의 쓸모는 여러 값이 한 처리를 공유하는 case 묶기이고, break 를 하나 빠뜨리면 다음 case 까지 실행되는 함정이 된다.
source: session
distilled: false
---

## 배운 개념

### 1. switch 는 라벨로 점프하고 아래로 흐른다

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
// 출력: 화 / 수 / 기타   (실제 실행으로 확인)
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

- **조건 검사는 진입할 때 딱 한 번.** 이후의 `case`·`default` 는 아무 의미 없는 표지판.
- 실행은 `break` 나 switch 블록 끝까지 **아래로 흘러 떨어진다** → fall-through.
- `break` 가 거의 모든 switch 예제에 붙어 있는 이유가 바로 이것.

### 2. fall-through 의 쓸모 — 여러 값이 한 처리를 공유

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

- 한 값은 case **하나**에만 맞는다. 같은 값의 case 를 두 번 쓰면 `duplicate case label` 컴파일 에러.
- 함정: `break` 하나 빠뜨리면 토요일이 **주말이면서 평일**이 된다.

### 면접 답변

> "switch 는 값에 맞는 case 라벨로 점프한 뒤, 그 아래 case 들은 다시 검사하지 않고 break 를 만날 때까지 아래로 계속 실행합니다. 이걸 fall-through 라고 합니다. 여러 값이 같은 처리를 공유할 때 case 를 묶는 데 쓸 수 있지만, break 를 빠뜨리면 의도하지 않은 case 까지 실행되는 버그가 됩니다."

## 오답·헷갈린 점

- **전**: 출력은 "화" 하나. "switch 는 case 의 모든 값을 찾아 맞는 곳의 코드를 실행하고, 모든 조건을 검사한 후 종료" → **후**: if-else 사슬 모델. 그 모델이면 `break` 가 필요 없다는 모순으로 정정. 맞는 **라벨로 점프 → 아래로 흐름 → break**.
- **전**: (정정 후 1차) 출력은 "화 기타". "위에서부터 내려오면서 조건에 맞다면 들러서 실행" → **후**: 진입 뒤에는 **어떤 라벨도 검사하지 않는다**. day 는 3 이 아닌데 "수"도 찍힌다 → 실제 실행 결과(3줄)로 확인.
- **전**: fall-through 의 쓸모 = "여러 케이스를 진행해야 될 때, 조건을 여러 개 충족하는 경우" → **후**: 한 값이 여러 case 를 충족하는 일은 없다(중복 case 컴파일 에러). 쓸모는 **여러 값 → 한 처리**.
- **전**: 주말 판별 코드를 case 마다 println 을 따로 쓰고 break 없이 작성 → 실행하니 day = 6 은 "주말 주말 기타", day = 1 은 8줄 → **후**: `case 6: case 7:` 로 묶고 처리 한 번 + `break`.
- 👍 묶은 코드의 추적 3개(day = 6, day = 3, break 삭제) 모두 정답 — 모델 전환 완료.

## Q&A

**Q. fall-through 란?**
A. switch 가 맞는 case 라벨로 점프한 뒤 break 나 블록 끝까지 아래 case 들을 검사 없이 계속 실행하는 것.

**Q. break 없는 switch 에서 day = 2 면?**
A. 화, 수, 기타. case 3 과 default 는 라벨이라 검사하지 않고 지나친다.

**Q. fall-through 를 일부러 쓰는 경우는?**
A. 여러 값이 같은 처리를 공유할 때 `case 6: case 7:` 처럼 묶는다.

## 다음 학습

- 문자열 switch 가 내부에서 `hashCode()` + `equals()` 로 동작하는 방식, null 이면 NPE
- 바이트코드 `tableswitch` vs `lookupswitch` (점프 테이블)
