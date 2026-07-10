# WRITING.md — 노트 작성 규약 (source of truth)

## 폴더 구조

```
01_Raw_Logs/<member>/YYYY-MM/YYYY-MM-DD-<주제슬러그>.md   # 학습 세션노트 (raw)
02_Refined/<주제슬러그>.md                                # 원자 개념노트 (evergreen)
INDEX.md                                                  # 02_Refined 색인
.review-state/<slug>.json                                 # 개인 복습 상태 (gitignore, 로컬)
.member                                                   # 내 멤버 핸들 (gitignore, 로컬)
```

- raw 도 공유 레포에 커밋된다. 멤버별 하위폴더(`01_Raw_Logs/<member>/`)를 벗어나 쓰지 말 것.
- 주제슬러그는 소문자 영문 kebab-case.

## Raw 노트 (01_Raw_Logs) frontmatter

```yaml
---
date: YYYY-MM-DD
type: raw
author: <member>
tags: [한/영 동의어 나열]
topic: 한 줄 주제
summary: 한두 문장 요약
source: session | brain:<uuid>
distilled: false
---
```

본문 섹션(순서 고정):

1. `## 배운 개념`
2. `## 오답·헷갈린 점` — 인터뷰/학습 중 틀렸다가 정정한 내용. 전→후 형태로.
3. `## Q&A`
4. `## 다음 학습`

코드·명령·mermaid는 원문 블록 그대로 유지.

## Refined 노트 (02_Refined) frontmatter

```yaml
---
type: refined
slug: <파일명과 동일한 kebab-case>
tags: [한/영 동의어 — 검색용, 풍부하게]
topic: 한 줄 주제
summary: 한두 문장 요약
contributors: [member1, member2]
source_refs: [외부 안정 URL만 — raw 경로·brain uuid 금지(dangling)]
updated: YYYY-MM-DD
---
```

- 개인 메타(`last_reviewed`/`confidence`)와 raw 추적은 로컬 `.review-state/<slug>.json` 사이드카에 (02에 넣지 말 것).

본문: 개념 정의 → 작동 원리 → 트레이드오프/한계 → 오답 코너(`> [!WARNING]`) → `## 복습 체크` 체크리스트 → `[[관련-slug]]` 링크.

## 하드 룰

- 대화 transcript 통째 덤프 금지 — 이해한 내용으로 재구성.
- **스크럽 게이트**: 회사 내부 실값(실 IP, 토큰, DB 행, 고객 PII, 내부 실코드)은 저장 전 일반화.
- 02_Refined 새 파일 만들기 전 git pull + tags grep으로 기존 노트 확인(중복 방지).
