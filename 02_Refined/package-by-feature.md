---
type: refined
slug: package-by-feature
tags: [package-by-feature, 기능별패키지, 패키지구조, package-by-layer, 레이어별패키지, 응집도, cohesion, 모듈화, project-structure]
topic: 기능별 패키지 구성 (package-by-feature)
summary: 패키지를 레이어(controller/service)가 아니라 기능(post/plan/notification) 단위로 묶어 응집도를 높이는 구조화 원칙.
contributors: [dongju]
source_refs: []
updated: 2026-07-10
---

# 기능별 패키지 구성 (package-by-feature)

## 정의

패키지를 기술 레이어(`controller/`, `service/`, `repository/`)가 아니라 **비즈니스 기능**(`post/`, `plan/`, `notification/`) 단위로 묶는 구조화 방식.

## 왜 필요한가

- **레이어별(package-by-layer)의 한계**: 기능이 늘수록 한 폴더가 비대해지고, 기능 하나를 수정하려면 세 폴더를 넘나들어야 한다.
- **핵심 원칙 = 응집도(cohesion)**: "같이 바뀌는 것들은 같이 둔다." 게시판 수정은 `post/` 안에서 끝난다.
- 기능 경계가 폴더 경계와 일치하므로, 특정 기능을 떼어 마이크로서비스로 분리할 때도 유리하다.

## 주의점

- 폴더만 나눈다고 결합이 끊기지 않는다 — 기능 모듈끼리 서로의 서비스를 직접 호출하면 장점이 무너진다. 모듈 간 통신은 [[spring-application-events]] 같은 간접 방식으로.

## 복습 체크

- [ ] 레이어별 구성의 한계를 "변경"의 관점에서 설명할 수 있다
- [ ] "같이 바뀌는 것들은 같이 둔다"가 어떤 원칙의 표현인지 말할 수 있다
- [ ] 기능별로 나눴을 때 모듈 간 직접 호출이 왜 위험한지 설명할 수 있다

관련: [[spring-application-events]]
