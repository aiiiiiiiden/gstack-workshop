# P5 — 댓글: Plan 모드로 합의를 먼저 하고, /review로 구멍을 메우기

> **재연 고지**: 이 편에 인용된 Plan 모드 대화와 `/review` 출력은 실제 세션의 로그를 바탕으로 요약·정리한 형태입니다. 지면 분량상 전체 출력을 그대로 옮기지 않았고, 결정·지적된 지점만 남겼습니다. 재현은 `git checkout v0.5` 이후 같은 프롬프트로 시도해 볼 수 있습니다.

## TL;DR

- 이번 편에서 만드는 것: 스레드 1단계 댓글 CRUD(중첩 없음), 작성자만 삭제할 수 있는 권한, 한 차례의 `/review` 이터레이션.
- 이번 편에서 해부하는 하네스 개념: **Plan 모드와 리뷰 사이클 패턴**. Plan 모드의 읽기 전용 성격이 어떻게 "설계 먼저" 규율을 강제하는지, ExitPlanMode(계획 종료 도구)가 실제로 무엇을 바꾸는지, 리뷰 사이클이라는 하네스 패턴이 어떤 축으로 코드를 읽는지. 이번 편이 해부에 쓰는 구체 인스턴스는 본 시리즈 표본의 `/review` 스킬입니다.
- 이 편이 끝나면 댓글 엔드포인트 두 개(`POST`/`DELETE`)가 통합 테스트를 통과하고, 남의 댓글 삭제는 403을 받습니다. Plan 모드 대화 한 덩어리와 `/review`가 지적한 한 이슈의 수정 흔적이 본문에 남습니다.

## 지난 편 요약

v0.4에서는 스레드 CRUD를 두 서브에이전트 병렬 호출로 구현했습니다. 이때 계약서 한 장(`/docs/contracts/threads.md`)이 두 작업자의 공통 진실 역할을 했고, 병합 과정에서 `createdAt` vs `created_at` 같은 충돌 세 개를 정리했습니다. 이를 통해 "서브에이전트는 계약이 있을 때만 빠르다"는 운영 원칙을 얻었습니다.

이번 편은 그 반대편에 있습니다. 댓글은 스레드보다 작은 기능이지만 권한 결이 미묘합니다. "자기 글만 삭제" 같은 규칙이 하나만 들어가도, 상태 전이를 그림으로 그려 두지 않으면 구현 중간에 길을 잃기 쉽습니다. 이번 편은 구현 전에 설계 합의를 먼저 하는 편입니다.

## 이번 편 목표

이 섹션은 이번 편의 흐름을 네 단계로 미리 그려 두고, 각 단계가 왜 그 순서여야 하는지를 설명합니다. 첫째, Plan 모드로 진입해서 댓글 기능의 스키마·엔드포인트·권한 규칙을 대화로 굳힙니다. 둘째, ExitPlanMode로 빠져나와 실제 구현을 합니다. 셋째, 리뷰 사이클을 돌려 구현을 점검합니다(본 시리즈 표본의 `/review` 스킬을 인스턴스로 씁니다). 넷째, 리뷰가 찾은 이슈 중 하나를 고치고, 그 수정의 전후를 본문에 남깁니다.

## Plan 모드로 진입해 설계 합의

이 섹션은 Plan 모드에 들어가 댓글 기능의 스키마와 권한 규칙을 문장으로 굳히는 과정을 기록합니다. Plan 모드는 `Shift+Tab` 단축키로 켜거나, 세션 시작 시 `--permission-mode plan`으로 명시할 수 있습니다. 모드 진입 후에는 화면 하단 상태줄에 `plan` 표시가 뜹니다.

Plan 모드에서 저는 다음 프롬프트로 대화를 시작했습니다.

```
댓글 기능을 추가할 건데, 먼저 설계만 합의하자.
- 스레드당 1단계 댓글만 허용한다(대댓글 없음).
- 작성자만 자기 댓글을 삭제할 수 있다.
- 엔드포인트 두 개: POST /threads/:id/comments, DELETE /comments/:id.
- 와이어 포맷은 기존 스레드와 동일하게 camelCase, 타임스탬프는 ISO-8601 UTC.
이 조건으로 스키마와 핸들러 시그니처를 잡아 줘. 코드는 쓰지 마.
```

모델은 Plan 모드의 규약상 파일을 편집할 수 없으므로, 대신 다음을 되돌려주었습니다.

```
계획:
1) db/migrations/003_comments.sql
   - comments(id bigserial, thread_id bigint FK, author_id bigint FK,
             body text, created_at timestamptz)
   - 인덱스: (thread_id, created_at DESC)
2) 핸들러 시그니처
   - POST /threads/:id/comments
     요청: { body: string }
     응답: 201 Comment
     체크: JWT 필수, 스레드 존재 확인
   - DELETE /comments/:id
     응답: 204 / 403(남의 댓글) / 404
     체크: JWT 필수, author_id == sub 확인
3) 계약서: /docs/contracts/comments.md (threads.md와 같은 경량 포맷)
```

이 단계에서 저는 두 가지를 더 물었습니다. 하나는 "삭제 시 소프트 삭제냐 하드 삭제냐"였고, 답은 v0.4와 일관되게 하드 삭제였습니다. 다른 하나는 "본문 길이 제한"이었고, 서버는 `≤ 2000 chars`로 잡고 클라이언트는 UX 차원에서만 강제하기로 했습니다. 서버가 권위를 갖는 규칙은 v0.3의 합의 그대로입니다.

대화가 정리되자 저는 모델에게 "이 계획을 `/docs/plans/P5-comments-plan.md`에 써 둬도 되겠지만, Plan 모드에서는 쓸 수 없으니 ExitPlanMode 후에 기록하자"고 말했습니다. 이 대화 블록 자체가 이번 편 Done When ②의 "Plan 모드 전사본"에 해당합니다.

## ExitPlanMode → 실제 구현

여기서는 ExitPlanMode 호출 직후 권한 모드가 어떤 상태로 복귀하고, 그 위에서 두 핸들러를 실제로 구현하는 과정을 다룹니다. Plan 모드를 빠져나오는 순간 권한 모드는 이전 상태로 복귀합니다. 이번 세션에서는 `acceptEdits`로 돌아왔습니다. 즉, 파일 편집은 확인 없이 허용되지만 Bash 실행은 여전히 확인을 거칩니다. 저는 ExitPlanMode 직후 위 계획을 그대로 `docs/plans/P5-comments-plan.md`에 저장했습니다. 이 파일이 v0.5 태그의 일부입니다.

구현 자체는 특별한 일이 아닙니다. `db/migrations/003_comments.sql`을 추가하고 `apps/dart_server/lib/handlers/comments.dart`에 두 핸들러를 넣었습니다. 핵심 로직만 발췌하면 이렇습니다.

```dart
Future<Response> createComment(Request req, String threadIdStr) async {
  final userId = req.context['userId'] as int;
  final threadId = int.parse(threadIdStr);
  final body = jsonDecode(await req.readAsString()) as Map;
  final text = (body['body'] as String).trim();
  if (text.isEmpty || text.length > 2000) {
    return Response(422, body: jsonEncode({'error': 'invalid body'}));
  }
  final row = await db.insertComment(threadId, userId, text);
  return Response(201, body: jsonEncode(rowToCamel(row)));
}

Future<Response> deleteComment(Request req, String idStr) async {
  final userId = req.context['userId'] as int;
  final id = int.parse(idStr);
  final comment = await db.findComment(id);
  if (comment == null) return Response(404);
  if (comment.authorId != userId) return Response(403);
  await db.deleteComment(id);
  return Response(204);
}
```

이 두 핸들러가 원하는 것은 명확합니다. 권한 검사가 응답 분기에 직접 드러나고, 응답 코드가 계약과 일치합니다. 통합 테스트에서 네 케이스(생성 201, 남의 댓글 삭제 403, 없는 댓글 삭제 404, 본인 댓글 삭제 204)가 초록으로 떨어졌습니다.

## 리뷰 사이클로 구멍 찾기

이 섹션은 리뷰 사이클 패턴을 한 번 돌려 구현을 점검하고, 발견된 이슈 하나를 고치는 과정을 기록합니다. 리뷰 사이클은 하네스 공통 패턴이고, 구체 인스턴스는 여러 형태를 띱니다. 다른 하네스에서도 리뷰 스킬은 대체로 같은 자리에 놓입니다. 여기서는 본 시리즈 표본의 `/review` 스킬을 인스턴스로 씁니다. 이 인스턴스는 프로젝트의 현재 변경 사항(diff 범위)을 대상으로 코드 품질·정확성·일관성을 훑으며, 별도 설정이 없으면 마지막 커밋 이후의 변경을 봅니다.

다음과 같이 실행했습니다.

```
/review
```

`/review`가 되돌려준 보고서는 다섯 개의 축으로 정리되어 있었습니다. 이 편에서는 **정확성** 축의 한 지적만 본문에 옮깁니다. 나머지 축(가독성, 테스트 커버리지 등)은 이 편의 해부 대상이 아닙니다.

```
정확성:
- apps/dart_server/lib/handlers/threads.dart:87
  목록 엔드포인트가 각 스레드의 댓글 수를 N+1 쿼리로 채우고 있습니다.
  스레드 20건을 반환할 때 댓글 카운트 쿼리가 20번 추가로 발사됩니다.
  수정 제안: SQL LEFT JOIN + COUNT(*) 서브쿼리, 또는 단일
  `comments` 집계 쿼리 후 메모리 조인.
```

저는 이 지적이 맞다는 것을 바로 확인했습니다. 스레드 목록 쿼리를 돈 뒤, 각 스레드마다 `SELECT COUNT(*) FROM comments WHERE thread_id = ?`를 돌리고 있었기 때문입니다. 실제 로컬에서 스레드 20건을 띄우면 21번의 쿼리가 발사되었습니다.

## 한 이슈 수정

다음 작업은 N+1 쿼리(한 번의 목록 조회 이후 각 행마다 추가 쿼리가 발사되는 패턴)를 단일 LEFT JOIN으로 교체하는 일입니다. 수정 방향은 LEFT JOIN 쪽을 선택했습니다. 메모리 조인이 더 유연하지만, 이번 스케일에서는 단일 SQL로 끝내는 편이 읽기 쉽습니다. 수정 전후는 다음과 같습니다.

```sql
-- 수정 전 (N+1을 유발)
SELECT id, author_id, title, body, created_at, updated_at
FROM threads
ORDER BY created_at DESC, id DESC
LIMIT $1;
-- 이후 각 행마다 SELECT COUNT(*) FROM comments WHERE thread_id = ?

-- 수정 후 (1회 쿼리)
SELECT t.id, t.author_id, t.title, t.body, t.created_at, t.updated_at,
       COALESCE(c.cnt, 0) AS comment_count
FROM threads t
LEFT JOIN (
  SELECT thread_id, COUNT(*) AS cnt
  FROM comments
  GROUP BY thread_id
) c ON c.thread_id = t.id
ORDER BY t.created_at DESC, t.id DESC
LIMIT $1;
```

수정 뒤 같은 목록 호출에서 쿼리는 한 번으로 줄었습니다. 통합 테스트가 여전히 초록색이었고, 응답 JSON에 `commentCount` 필드가 추가되어 Flutter 쪽 모델도 한 줄 늘어났습니다. 이 수정은 `apps/flutter_client/lib/features/threads/thread.dart`에 `commentCount: json['commentCount'] ?? 0`를 더하는 것으로 끝났습니다.

## 해부: Plan 모드와 리뷰 사이클 패턴

> **해부: Plan 모드와 리뷰 사이클**
>
> **Plan 모드의 제약**: 네 권한 모드 중 하나입니다. 파일 읽기·탐색·도구의 읽기 전용 호출은 허용되지만 Edit/Write/Bash 같은 상태 변경 도구는 모두 차단됩니다. 이 제약이 설계를 "문장으로 끝내게" 강제합니다. 모드 내에서 모델은 변경을 제안만 할 뿐 실행하지 못합니다.
>
> **ExitPlanMode가 바꾸는 것**: ExitPlanMode 도구가 호출되면 사용자는 계획을 승인·거부·수정 요청할 수 있고, 승인 즉시 권한 모드가 이전 모드(보통 default 또는 acceptEdits)로 복귀합니다. 복귀 이후에만 Edit/Write/Bash 도구가 다시 열립니다. 즉, "계획 합의 → 승인 → 실행"이라는 한 사이클이 도구 호출 수준에서 강제됩니다.
>
> **리뷰 사이클의 시선**: 리뷰 스킬의 형태와 상관없이, 이 패턴은 진단 축을 고정하는 쪽으로 수렴합니다. 본 시리즈가 쓴 `/review` 인스턴스의 경우 정확성, 일관성, 테스트 커버리지, 가독성, 프로젝트 고유 규약 위반 다섯 축을 잡습니다. 범위는 기본적으로 "현재 변경 사항"(staged + 마지막 커밋 이후의 변경)입니다. 이 범위 기본값 덕분에 대형 리팩터 직전에 돌리면 진단이 발산하지 않습니다.
>
> **Plan 모드와 리뷰 사이클의 짝**: 둘은 구현 양 끝에서 점검을 건다는 점에서 짝입니다. Plan 모드는 시작 지점에서 "방향"을 점검하고, 리뷰 사이클은 종료 지점에서 "틈"을 점검합니다. 한쪽만 써도 부분적으로 유효하지만, 두 번의 점검 사이에 구현을 끼워 넣는 패턴이 이번 편의 기본형입니다.

이 네 가지, 즉 Plan 모드의 제약, ExitPlanMode의 복귀, 리뷰 사이클의 시선, 둘의 짝을 묶어 두면 "설계 먼저"가 의지의 문제가 아니라 도구 경계의 문제로 내려옵니다. 의지는 흔들리지만 도구 경계는 흔들리지 않습니다.

## Done When

- [ ] 남의 댓글을 삭제하는 요청이 `403`을 받습니다. `test/comments_integration_test.dart`의 네 케이스(201/403/404/204)가 모두 초록색입니다.
- [ ] `docs/plans/P5-comments-plan.md`에 Plan 모드에서 합의한 계획이 보존되어 있고, 이 편 본문에도 그 전사본이 인용되어 있습니다.
- [ ] `/review`가 지적한 이슈 중 최소 하나를 고친 흔적이 본문에 남아 있습니다. 이번 편에서는 스레드 목록의 N+1 쿼리가 그 대상입니다.

## 마치며

이번 편에서는 댓글 기능을 Plan 모드 합의와 리뷰 사이클 점검의 샌드위치 사이에 넣어 구현했습니다. Plan 모드의 읽기 전용 성격이 설계 단계를 문장으로 끝내게 했고, ExitPlanMode의 복귀가 구현 시작의 신호가 되었습니다. 또한 리뷰 사이클이 지적한 N+1 쿼리를 LEFT JOIN으로 수정하면서, 리뷰가 단순한 점검이 아니라 구현의 한 단계라는 점을 확인했습니다. 본 시리즈가 쓴 리뷰 인스턴스는 `/review` 스킬이었고, 다른 하네스라면 같은 자리에 해당 인스턴스의 리뷰 스킬이 들어갈 수 있습니다.

이 편의 요지는 "Plan 모드가 좋다"가 아닙니다. 요지는 "모드가 바뀌면 허용되는 도구가 바뀐다"는 점입니다. 이 당연한 사실이 왜 규율로 작동하냐면, 의지와 달리 도구 경계는 흔들리지 않기 때문입니다. 설계 단계에서 Edit가 닫혀 있으면, 모델은 "일단 써 보자"로 미끄러질 수 없습니다. 이에 따라 설계는 문장으로 닫히고, 구현은 방향을 잃지 않습니다.

다음 편 v0.6에서는 좋아요 기능을 붙이면서 훅의 한계를 드러내는 장면을 다룹니다. 파괴적 SQL(`DROP TABLE`) 패턴을 PreToolUse 훅으로 차단하는 구현과, 같은 훅이 `docker exec`나 ORM 경로에서는 우회된다는 사실을 함께 기록합니다. 훅이 완벽한 샌드박스가 아니라 "자주 저지르는 실수를 위한 안전망"임을 드러내는 것이 그 편의 해부 축입니다.

## 이 편 끝난 상태

v0.5 태그가 가리키는 상태입니다.

- `db/migrations/003_comments.sql`: `comments` 테이블 + 인덱스 `(thread_id, created_at DESC)`
- `apps/dart_server/`: 두 엔드포인트 + 권한 분기 + 통합 테스트 네 케이스
- `apps/flutter_client/`: 댓글 목록·작성 UI + `comment_repository.dart`
- `docs/contracts/comments.md`: 경량 계약
- `docs/plans/P5-comments-plan.md`: Plan 모드 합의 전사본
- `apps/dart_server/lib/handlers/threads.dart`: N+1 수정된 목록 쿼리

## 참조

- Claude Code 권한 모드 문서: https://docs.claude.com/en/docs/claude-code/iam
- 본 시리즈가 쓴 `/review` 인스턴스(gstack 스킬 목록의 `review/SKILL.md`): https://github.com/garrytan/gstack
- Postgres LEFT JOIN + GROUP BY: https://www.postgresql.org/docs/16/tutorial-join.html
