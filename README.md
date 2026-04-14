# gstack-workshop

A Korean blog series that dissects the Claude Code + gstack **harness** (skills, hooks, settings, subagents) using a developer-community MVP as its stage — with a companion reference implementation git-tagged at the end of each post.

- **Stack**: Flutter + Dart server (shelf) + PostgreSQL + Docker
- **Blog language**: Korean (code comments are English)
- **Infra-doc language** (this README, `docs/series/DESIGN.md`, `docs/contracts/`): English
- **Series source**: [`/docs/series/`](./docs/series/)
- **Design doc**: [`/docs/series/DESIGN.md`](./docs/series/DESIGN.md)

## Series Outline (tentative)

| Tag  | Title (Korean, to be drafted)            | Harness focus                               |
|------|------------------------------------------|---------------------------------------------|
| v0.0 | 프롤로그 — 하네스란 무엇인가              | harness vs Claude Code vs gstack            |
| v0.1 | 환경 세팅 — 스택과 gstack 첫 터치         | how a skill loads                           |
| v0.2 | 회원가입 — 커스텀 스킬                    | skill structure end-to-end                  |
| v0.3 | 로그인 — 훅·권한 모델                     | PreToolUse / PostToolUse, settings layering |
| v0.4 | 스레드 — 병렬 subagent                    | Agent tool, self-contained prompts          |
| v0.5 | 댓글 — Plan mode + /review                | plan mode, review cycle                     |
| v0.6 | 좋아요 — 파괴 SQL 차단 훅                 | hook limits, safety-net framing             |
| v0.7 | 에필로그 — 멘탈 모델                      | whole-system overview                       |

## Reader Quickstart

To reproduce the state at any post:

```bash
git clone git@github.com:aiiiiiiiden/gstack-workshop.git
cd gstack-workshop
git checkout v0.N        # N = 0..7
docker-compose up        # MVP runs locally (from v0.1 onward)
```

## Tag Rule

Tags are **append-only**. Bug fixes for earlier posts land in the **next** tag; prior tags are never force-pushed. Each tag is the honest snapshot of that post's ending state.

## Repo Layout

```
.
├── README.md                     # this file (English)
├── docker-compose.yml            # from v0.1
├── apps/
│   ├── dart_server/              # shelf-based HTTP server
│   └── flutter_client/           # Flutter app
├── db/                           # Postgres schema / migrations
├── .claude/                      # Claude Code local settings (gitignored) + project skills
└── docs/
    ├── series/                   # blog posts 00..07 (Korean)
    │   └── DESIGN.md             # series design spec (English)
    ├── contracts/                # API contracts (English, TypeScript-style interfaces)
    └── assets/<post-number>/     # screenshots for each post
```

## License

TBD — decided at the P3 checkpoint together with the publication platform choice.
