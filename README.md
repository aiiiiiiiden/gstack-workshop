# gstack-workshop

개발자 커뮤니티 MVP를 무대로 Claude Code + gstack 하네스(harness)를 해부하는 한국어 블로그 시리즈 + 동반 참조 구현.

- **스택**: Flutter + Dart 서버(shelf) + PostgreSQL + Docker
- **언어**: 한국어 (코드 주석은 영문)
- **시리즈 원본**: [`/docs/series/`](./docs/series/)
- **설계 문서**: [`/docs/series/DESIGN.md`](./docs/series/DESIGN.md)

## 시리즈 구성 (잠정)

| 태그 | 제목 | 핵심 하네스 주제 |
|------|------|------------------|
| v0.0 | 프롤로그 — 하네스란 무엇인가 | 하네스 vs Claude Code vs gstack |
| v0.1 | 환경 세팅 — 스택과 gstack 첫 터치 | 스킬 로드 메커니즘 |
| v0.2 | 회원가입 — 커스텀 스킬 | 스킬 구조 전반 |
| v0.3 | 로그인 — 훅·권한 모델 | PreToolUse/PostToolUse, settings 계층 |
| v0.4 | 스레드 — 병렬 subagent | Agent tool, self-contained prompt |
| v0.5 | 댓글 — Plan mode + /review | plan mode, review 싸이클 |
| v0.6 | 좋아요 — 파괴 SQL 차단 훅 | 훅의 한계와 안전망 관점 |
| v0.7 | 에필로그 — 멘탈 모델 | 전체 흐름 정리 |

## 독자 사용법

특정 편 상태를 재현하고 싶다면:

```bash
git clone git@github.com:aiiiiiiiden/gstack-workshop.git
cd gstack-workshop
git checkout v0.N   # N = 0~7
docker-compose up   # MVP 로컬 구동 (v0.1 이후)
```

## 태그 규칙

태그는 **append-only**. 이전 편 버그 수정은 다음 태그에 포함되며, 과거 태그를 force-push로 덮지 않음. 각 태그는 해당 편을 쓸 당시의 솔직한 스냅샷.

## 라이선스

추후 결정 (P3 완료 시점에 공개 출판처와 함께 결정).
