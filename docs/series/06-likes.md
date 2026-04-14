# P6 — 좋아요: 훅이 막을 수 있는 것과 막지 못하는 것

> **재연 고지**: 이 편의 `DROP TABLE likes;` 직전 장면은 실제 세션에서 staged 명령을 가로챈 사건을 기반으로 합니다. 다만 본문에서는 발생 순서를 단순화하고, 차단 메시지의 문구를 가독성을 위해 다듬었습니다. 같은 훅과 같은 명령을 재현하면 같은 결과가 나옵니다(`git checkout v0.6`).

## TL;DR

- 이번 편에서 만드는 것: 좋아요 토글 두 엔드포인트(`POST`/`DELETE /threads/:id/like`)와 카운터, 같은 사용자의 더블클릭에 견디는 유니크 제약, 그리고 `DROP TABLE` 패턴을 잡는 PreToolUse 훅 한 줄.
- 이번 편에서 해부하는 하네스 개념: **훅의 한계와 설계 원칙**. 훅은 완벽한 샌드박스(sandbox)가 아닙니다. 어떤 경로는 막고, 어떤 경로는 그대로 통과합니다. 이 편은 그 경계를 명시적으로 그리는 데 시간을 씁니다.
- 이 편이 끝나면 좋아요가 동시 클릭 두 번에도 중복 행을 남기지 않고, `psql -c 'DROP TABLE likes;'`가 차단됩니다. 동시에 본문은 "그래도 `docker exec`로는 통과한다"는 사실을 숨기지 않고 적습니다.

## 지난 편 요약

v0.5에서는 댓글 기능을 Plan 모드 합의와 `/review` 점검 사이에 끼워 구현했습니다. Plan 모드의 읽기 전용 성격이 설계 단계를 문장으로 닫게 했고, ExitPlanMode 이후 권한 모드 복귀가 구현 시작의 신호가 되었습니다. 또한 `/review`가 지적한 N+1 쿼리를 LEFT JOIN으로 고치며, 리뷰가 점검이 아니라 구현의 한 단계라는 점을 확인했습니다.

이번 편은 구조의 한 축을 더 다룹니다. 훅이 막는 것과 막지 못하는 것의 경계입니다. 이전 편(v0.3)에서 PreToolUse 훅으로 `.env`가 `git add` 라인에 끼지 못하게 막았습니다. 이번에는 같은 종류의 훅을 한 가지 더 늘리되, **이 훅이 어디서 우회되는지**를 같은 본문에서 드러냅니다.

## 이번 편 목표

이 섹션은 좋아요 기능 구현과 두 갈래 방어선을 미리 그려 둡니다. 한 갈래는 데이터 모델 수준의 방어, 다른 한 갈래는 명령 실행 수준의 방어입니다. 첫째, 좋아요 토글 두 엔드포인트와 유니크 제약을 만들어 더블클릭 경쟁 조건을 데이터 계층에서 봉합합니다. 둘째, `DROP TABLE` 같은 파괴적 SQL이 `psql` 명령으로 발사되기 직전 PreToolUse 훅이 가로채도록 합니다. 마지막으로 이 훅이 통과시키는 경로를 본문에 그대로 적습니다.

## 좋아요 토글과 유니크 제약

이 섹션은 좋아요 두 엔드포인트와 데이터 모델을 잡고, 동시 요청에서 중복 행이 생기지 않도록 유니크 제약을 거는 과정을 기록합니다. 좋아요는 카운트가 아니라 **관계**입니다. 즉, "사용자 A가 스레드 X를 좋아한다"는 한 행이 존재하거나 존재하지 않거나 둘 중 하나입니다. 카운트는 이 행들의 집계입니다.

```sql
-- db/migrations/004_likes.sql
CREATE TABLE likes (
  user_id   bigint NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  thread_id bigint NOT NULL REFERENCES threads(id) ON DELETE CASCADE,
  created_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (user_id, thread_id)
);
CREATE INDEX likes_thread_idx ON likes (thread_id);
```

기본 키가 `(user_id, thread_id)` 복합이라는 점이 핵심입니다. 같은 사용자가 같은 스레드를 두 번 좋아요 시도하면, 두 번째 INSERT는 Postgres가 키 충돌로 거절합니다. 이 거절을 핸들러가 `ON CONFLICT DO NOTHING`으로 받아 멱등(idempotent)한 응답으로 변환합니다.

```dart
// apps/dart_server/lib/handlers/likes.dart (발췌)
Future<Response> like(Request req, String threadIdStr) async {
  final userId = req.context['userId'] as int;
  final threadId = int.parse(threadIdStr);
  await db.execute(
    'INSERT INTO likes (user_id, thread_id) VALUES (\$1, \$2) '
    'ON CONFLICT DO NOTHING',
    [userId, threadId],
  );
  return Response(204);
}
```

좋아요 해제 핸들러는 대칭 구조입니다. 같은 키로 행을 삭제하고, 행이 없어도 결과가 같으므로 똑같이 멱등합니다.

```dart
Future<Response> unlike(Request req, String threadIdStr) async {
  final userId = req.context['userId'] as int;
  final threadId = int.parse(threadIdStr);
  await db.execute(
    'DELETE FROM likes WHERE user_id = \$1 AND thread_id = \$2',
    [userId, threadId],
  );
  return Response(204);
}
```

행이 이미 없어도 `DELETE`는 0건 삭제로 끝나므로 응답이 동일합니다. 멱등 보장이 양쪽에서 동일한 셈입니다.

카운터는 별도 컬럼으로 두지 않습니다. 스레드 목록 쿼리에서 P5의 LEFT JOIN 패턴을 그대로 빌려 `likes` 집계를 같이 실어 보내거나, 상세 화면에서만 `SELECT COUNT(*) FROM likes WHERE thread_id = $1`을 한 번 더 부릅니다. 카운터 컬럼을 따로 두면 좋아요/해제마다 동기화 비용이 생기는데, 이번 스케일에서는 집계 한 번이 더 정직합니다.

같은 사용자가 동시에 두 번 클릭해 두 요청이 거의 동시에 도달해도, 두 INSERT 중 하나는 키 충돌로 무시됩니다. 트랜잭션이나 `SELECT FOR UPDATE`를 동원하지 않고 데이터 모델만으로 경쟁 조건을 봉합한 셈입니다. 락 기반 처리가 필요한 더 일반적인 경쟁 조건 패턴은 [Postgres 문서의 동시성 챕터](https://www.postgresql.org/docs/16/mvcc.html)로 미룹니다. 이 편의 범위는 좋아요 한 건의 멱등성까지입니다.

통합 테스트에서는 같은 사용자로 두 개의 future를 동시에 발사한 뒤, `likes` 테이블에서 `(user_id, thread_id)` 행 수가 정확히 1인지를 확인하는 케이스를 추가했습니다. 두 요청 모두 `204`를 받고 행은 하나만 남습니다. Done When ①이 여기서 채워집니다.

## DROP TABLE 직전 장면

이 섹션은 `DROP TABLE likes;` 명령이 `psql`로 발사되기 직전, PreToolUse 훅이 가로채는 장면을 재현합니다.

저는 좋아요 통합 테스트가 이상한 상태로 떨어진 디버깅 중이었습니다. 테스트가 남긴 더미 데이터를 한 번에 비우고 싶었고, 모델에게 "테이블을 비워 줘"라고 요청했습니다. 모델은 다음 명령을 제안했습니다.

```bash
docker exec -i workshop-postgres \
  psql -U workshop -d workshop -c 'DROP TABLE likes;'
```

이 명령은 테이블의 행이 아니라 테이블 구조 자체를 제거합니다. `TRUNCATE`와 `DROP TABLE`은 의도가 다릅니다. 전자는 행만 비우고 후자는 테이블 자체를 삭제합니다. 비슷해 보이는 한 단어 차이가 마이그레이션 상태를 망가뜨리는 분기점이 됩니다. 모델이 두 단어를 혼동한 순간, 이 명령은 도구 호출 큐로 들어갔습니다.

이 시점에 PreToolUse 훅이 동작했습니다. v0.3에서 만들어 둔 `deny-env-in-git-add.sh`와 같은 자리에, 이번에는 `deny-destructive-sql.sh`가 추가되어 있었습니다 — P3에서 git 명령에 적용한 패턴 매칭 방식을 파괴적 SQL 경로로 확장한 것입니다. 훅 스크립트는 다음과 같이 짧습니다.

```bash
#!/usr/bin/env bash
# .claude/hooks/deny-destructive-sql.sh
# stdin 으로 도구 호출 JSON 을 받습니다.
input="$(cat)"
cmd="$(echo "$input" | jq -r '.tool_input.command // empty')"

# psql 경로의 파괴적 SQL 패턴
if echo "$cmd" | grep -Eqi 'psql.*-c.*\b(DROP[[:space:]]+TABLE|TRUNCATE|DELETE[[:space:]]+FROM[[:space:]]+\w+[[:space:]]*;?$)'; then
  echo '{"decision":"block","reason":"파괴적 SQL은 psql 직접 실행이 아니라 마이그레이션 파일로 처리하세요."}'
  exit 0
fi
echo '{}'
```

`.claude/settings.json`의 `hooks` 항목에는 다음과 같이 `Bash` 도구 호출에 대한 PreToolUse로 등록되어 있습니다(v0.3에서 등록한 항목 옆에 한 줄 추가).

```json
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "Bash", "hooks": [{ "type": "command", "command": ".claude/hooks/deny-env-in-git-add.sh" }] },
      { "matcher": "Bash", "hooks": [{ "type": "command", "command": ".claude/hooks/deny-destructive-sql.sh" }] }
    ]
  }
}
```

훅이 `decision: "block"`을 돌려주자 도구 호출은 실행되지 못하고, 이유 문구가 모델 컨텍스트에 추가됐습니다. 모델은 그 자리에서 명령을 바꿔 `TRUNCATE TABLE likes;`를 제안했고, 이번에는 훅이 통과시켰습니다. Done When ②가 여기서 채워집니다.

## 통과시키는 경로를 본문에 적기

이 섹션은 위 훅이 막지 못하는 경로를 명시적으로 정리합니다. 이 정리는 안전망의 정직한 사용 설명서에 해당합니다.

- **`docker exec` 직접 호출 우회**: 위 훅의 정규식은 `psql.*-c.*DROP TABLE` 패턴을 잡습니다. 하지만 `docker exec -it workshop-postgres bash`로 셸에 들어간 뒤 그 안에서 `psql` 후 `DROP TABLE`을 입력하는 경로는 도구 호출 시점에 명령 문자열에 패턴이 없으므로 통과합니다.
- **ORM(Object-Relational Mapper)·드라이버 경로**: Dart 코드가 `db.execute('DROP TABLE likes;')`를 호출하면 이는 Bash 도구를 거치지 않습니다. 따라서 PreToolUse(matcher: Bash)는 발화하지 않습니다.
- **다른 파괴적 패턴**: 이번 정규식은 `DROP TABLE`, `TRUNCATE`, `DELETE FROM <table>;` 세 형태만 잡습니다. `ALTER TABLE … DROP COLUMN`이나 `UPDATE … SET` 같은 변형은 통과합니다. 이는 의도된 좁은 범위입니다. 패턴을 넓히면 정상 작업까지 막혀 거짓 양성이 늘어납니다.

이 세 우회 경로를 본문에 명시하는 이유는 단순합니다. 안전망을 "샌드박스"라고 부르는 순간 사용자가 그 안에서 안심하기 때문입니다. 안심은 사고를 부릅니다. 안전망은 자주 저지르는 실수 한 종류만을 잡고, 그 한 종류를 잡는다는 점만 약속합니다. Done When ③이 여기서 채워집니다.

## 해부: 훅의 한계와 설계 원칙

이 섹션은 훅이 개입하는 경계와 그 경계가 의도된 설계임을 다섯 가지 원칙으로 정리합니다.

> **해부: 훅이 막는 것과 막지 못하는 것**
>
> **훅은 도구 호출의 매개에 끼어듭니다, 행위 자체에 끼어들지 않습니다**: PreToolUse는 모델이 도구를 호출하려는 그 순간에 발화합니다. 따라서 도구 호출 형태로 표현된 행위만 검사 대상입니다. 모델이 코드를 직접 실행하지 않거나, 코드 안에서 일어나는 부수 효과(예: ORM이 발사하는 SQL)는 훅의 시야 밖입니다.
>
> **matcher가 시야의 폭을 결정합니다**: `matcher: "Bash"`는 Bash 도구 호출만 봅니다. `matcher: "Edit|Write"`는 파일 편집을 봅니다. `matcher: "*"`는 전부 보지만 그만큼 발화 빈도와 거짓 양성이 늘어납니다. 시야를 좁게 잡고 정규식을 좁게 짜는 편이 운영 비용이 적습니다.
>
> **정규식의 좁음은 약점이 아니라 정책입니다**: 이번 훅은 `psql.*-c.*DROP TABLE` 같은 한정된 패턴만 잡습니다. 패턴을 넓히면 거짓 양성으로 정상 작업이 자주 멈추고, 사용자는 결국 훅을 비활성화합니다. "자주 저지르는 한 가지를 정확히 잡는다"가 안전망 설계의 핵심입니다.
>
> **샌드박스가 아니라는 사실을 본문에 적습니다**: 우회 경로(docker exec 셸, ORM, 변형 SQL)를 같은 본문에 적어 두면 사용자는 훅을 "절대 안전"이 아닌 "잦은 실수에 대한 안전망"으로 인식합니다. 이 인식 차이는 큽니다. 절대 안전이라고 믿으면 사용자는 위험한 명령을 망설임 없이 발사하고, 안전망이라 알면 발사 전에 한 박자 멈춥니다.
>
> **샌드박스가 정말 필요하다면 권한 모드와 격리 환경을 함께 씁니다**: 훅의 한계가 신경 쓰이는 작업이라면 `bypassPermissions` 같은 강한 모드를 피하고, Docker 컨테이너나 worktree 같은 외부 격리 경계를 함께 사용합니다. 훅 한 층으로 모든 위험을 막으려는 설계는 무리입니다.

이 다섯 줄 — 매개의 위치, matcher의 시야, 정규식의 좁음, 본문 명시, 격리 경계와의 짝 — 을 묶어 두면 훅은 "막아 주는 마법"이 아니라 "한정된 안전망"으로 자리 잡습니다. 한정의 명시는 신뢰의 조건입니다.

## Done When

- [ ] 같은 사용자가 동시에 두 번 좋아요 요청을 보내도 `likes` 테이블에 한 행만 남습니다. `test/likes_concurrent_test.dart`가 이 케이스를 초록색으로 통과합니다.
- [ ] `psql -c 'DROP TABLE likes;'` 패턴의 명령이 PreToolUse 훅에 의해 차단됩니다. 차단 사유 문구가 모델 컨텍스트에 들어와 모델이 자발적으로 `TRUNCATE`로 전환하는 흐름이 본문에 재현되어 있습니다.
- [ ] 본문에 훅이 통과시키는 경로(`docker exec` 셸, ORM 호출, 변형 SQL)가 명시적으로 적혀 있습니다. 이 항목이 없으면 안전망이 샌드박스로 오인됩니다.

## 마치며

이번 편에서는 좋아요 기능을 데이터 모델 수준의 멱등성으로 봉합하고, `DROP TABLE` 패턴을 잡는 PreToolUse 훅을 한 줄 늘렸습니다. 또한 그 훅이 통과시키는 경로(`docker exec` 셸, ORM, 변형 SQL)를 같은 본문에 적어, 안전망의 폭을 정직하게 그렸습니다.

이 편의 요지는 "훅으로 안전을 만든다"가 아닙니다. 요지는 "훅의 한계를 본문에 적어야 안전이 신뢰가 된다"는 점입니다. 안전망의 폭을 숨기면 사용자는 그 안을 샌드박스로 오해하고, 오해는 가장 위험한 명령을 가장 가벼운 손가락으로 발사하게 만듭니다. 이에 따라 안전망의 경계를 같은 본문에 적는 행위 자체가, 그 경계를 설계 결정으로 격상시키는 과정이 됩니다.

다음 편 v0.7은 에필로그입니다. 여섯 편을 가로지르는 멘탈 모델을 한 장으로 정리하고, 시리즈가 의도적으로 다루지 않은 영역(MCP, Memory, ToolSearch, Learnings, OpenClaw 호환)을 한 줄씩 짚습니다. 그리고 독자가 이 시리즈 다음에 시도해 볼 만한 세 가지 실험을 제안하며 시리즈를 닫습니다.

## 이 편 끝난 상태

v0.6 태그가 가리키는 상태입니다.

- `db/migrations/004_likes.sql`: `likes` 테이블(복합 PK + 인덱스)
- `apps/dart_server/`: 좋아요 토글 두 핸들러 + 멱등 처리 + 동시성 통합 테스트
- `apps/flutter_client/`: 좋아요 버튼 토글 UI + 카운트 표시
- `.claude/hooks/deny-destructive-sql.sh`: 파괴적 SQL 패턴 차단 훅
- `.claude/settings.json`: PreToolUse 훅에 한 줄 추가
- 본문에 차단 장면과 우회 경로 목록

## 참조

- Postgres 동시성과 MVCC: https://www.postgresql.org/docs/16/mvcc.html
- Postgres `ON CONFLICT` 절: https://www.postgresql.org/docs/16/sql-insert.html#SQL-ON-CONFLICT
- Claude Code Hooks 공식 문서: https://docs.claude.com/en/docs/claude-code/hooks
