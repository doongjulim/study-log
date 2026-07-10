# PIPELINE.md — 학습위키 공유 모델

```
학습 세션 (study-interview 등)
   │
   ▼  study-capture (STEP 1)
01_Raw_Logs/<member>/YYYY-MM/*.md     ← 멤버별 세션노트, 공유 레포에 커밋
   │
   ▼  study-distill (STEP 2, raw 3개+ 쌓이면 권장)
02_Refined/<주제슬러그>.md             ← 원자 개념노트(evergreen), 멤버 공동 소유
   │
   ├─ study-recall  : 질문 답변 전 기존 노트 먼저 검색
   └─ study-review  : 능동 인출 퀴즈 복습 (.review-state/ 는 개인 로컬)
```

## 공유 규약

- raw / refined 모두 이 레포로 공유된다. raw 는 멤버별 폴더라 충돌 없음.
- 02_Refined 는 공동 소유: 같은 개념이면 새 파일 대신 기존 노트를 enrich/정정 (3-case 병합).
- 커밋: 각자 작업 후 직접 커밋. 원격 연결 시 push 전 pull --rebase.
- distill 후 해당 raw 의 frontmatter `distilled: true` 로 갱신.
