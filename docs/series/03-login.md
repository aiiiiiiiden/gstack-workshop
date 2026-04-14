# P3 — 로그인: 훅으로 비밀키 유출을 막고, 권한 모델을 뜯어봅니다

## TL;DR

- 이번 편에서 만드는 것: `POST /login` 엔드포인트, JWT(JSON Web Token) 발급·검증, Flutter 세션 관리, 그리고 `.env` 파일이 실수로 git에 스테이징되는 순간을 차단하는 PreToolUse 훅 하나.
- 이번 편에서 해부하는 하네스 개념: **훅 + 설정 계층 + 권한 모드** 세 가지를 한 편에 묶어 다룹니다. 이 세 개념은 서로 맞물려 돌아가므로 분리할 수 없습니다.
- 이 편 끝나면 로그인이 E2E로 돌아가고, `.env`가 포함된 `git add`가 자동으로 멈춥니다.

> **재연 고지**: 이 편의 "`git add .`로 `.env`를 스테이징할 뻔한 순간"은 시리즈 설계 시점에 준비된 재연 시나리오입니다. 실제 작성 과정에서 유사한 장면이 목격되기도 했지만, 이 글의 해당 대목은 재연이 포함되어 있음을 미리 알립니다.

## 지난 편 요약

v0.2에서는 회원가입을 E2E로 붙이고, 반복되는 세 파일(마이그레이션 + Dart 모델 + Flutter 모델)을 한 번에 생성하는 `/scaffold-feature` 커스텀 스킬을 작성했습니다. 또한 `SKILL.md`의 구조 — 프런트매터, 본문, 재귀 금지, 디렉토리 관례 — 를 해부했습니다.

이번 편에서는 로그인을 붙이며, 이전까지는 눈에 보이지 않던 기제 두 가지가 전면에 등장합니다. 하나는 훅입니다. 다른 하나는 권한 모드와 설정 계층입니다. 세 개념 모두 "모델이 도구를 호출하는 순간"을 둘러싸고 있다는 점에서 같은 층에 속합니다.

## 이번 편 목표

로그인 E2E를 붙이는 것은 기계적인 작업입니다. 이번 편의 진짜 목표는 그 과정에서 발생하는 위험 장면 하나 — `.env` 파일이 실수로 원격 저장소로 밀려 나가는 경우 — 를 기술적 기제로 차단하는 것입니다. 그리고 그 기제를 "왜 여기에, 어떤 타이밍으로, 어느 층의 설정으로" 놓는지를 뜯어봅니다.

## 로그인 엔드포인트

이 섹션은 `POST /login` 핸들러를 구현합니다. 요청 본문에서 `email`과 `password`를 받아 bcrypt로 검증한 뒤, 유효하면 JWT를 발급합니다. 회원가입과 대칭 구조입니다.

```dart
// apps/dart_server/bin/server.dart (발췌)
Future<Response> _login(Request req) async {
  final body = jsonDecode(await req.readAsString()) as Map<String, dynamic>;
  final email = (body['email'] as String).trim().toLowerCase();
  final password = body['password'] as String;

  final rows = await db.execute(
    Sql.named('SELECT id, password FROM users WHERE email = @e'),
    parameters: {'e': email},
  );
  if (rows.isEmpty) return Response(401, body: 'invalid');

  final id = rows.first[0] as int;
  final hash = rows.first[1] as String;
  if (!BCrypt.checkpw(password, hash)) return Response(401, body: 'invalid');

  final jwt = JWT(
    {'sub': id, 'email': email},
    // 짧은 만료를 의도적으로 씁니다. 만료 검증을 Done When ①에서 요구하기 때문입니다.
  ).sign(SecretKey(_jwtSecret), expiresIn: const Duration(minutes: 15));
  return Response.ok(jsonEncode({'token': jwt}),
      headers: {'content-type': 'application/json'});
}
```

만료 시간을 15분으로 짧게 잡는 이유는 Done When ①에서 "만료된 토큰을 거부한다"를 검증하기 위함입니다. 운영 환경에서는 이 값이 더 길거나 리프레시 토큰과 조합되지만, 이 시리즈는 MVP 범위에서 단일 토큰만 다룹니다. 또한 토큰 검증은 `verify` 헬퍼를 별도로 두어, 이후 편의 스레드·댓글 핸들러에서 재사용합니다.

Flutter 쪽은 로그인 성공 시 받은 JWT를 `shared_preferences`에 저장합니다. 이 선택은 의도적인 절충입니다. 실제 앱이라면 `flutter_secure_storage`를 쓰는 편이 안전하지만, 이 시리즈는 secure storage의 플랫폼별 이슈까지 다루지 않습니다. 개발 편의를 우선하고, 보안 보강은 에필로그에서 다음 단계로 남깁니다.

코드는 회원가입 화면과 대칭 구조입니다. 차이는 저장 로직뿐입니다.

```dart
// apps/flutter_client/lib/features/login/login_screen.dart (발췌)
Future<void> _submit() async {
  if (!_formKey.currentState!.validate()) return;
  final res = await http.post(
    Uri.parse('$kApiBase/login'),
    headers: {'content-type': 'application/json'},
    body: jsonEncode({'email': _email.text.trim(), 'password': _password.text}),
  );
  if (res.statusCode != 200) {
    setState(() => _status = 'fail ${res.statusCode}');
    return;
  }
  final token = (jsonDecode(res.body) as Map<String, dynamic>)['token'] as String;
  final prefs = await SharedPreferences.getInstance();
  await prefs.setString('jwt', token);
  if (!mounted) return;
  Navigator.of(context).pushReplacement(
    MaterialPageRoute(builder: (_) => const HomeScreen()),
  );
}
```

이후 편의 보호된 엔드포인트 호출에서는 `prefs.getString('jwt')`로 토큰을 꺼내 `Authorization: Bearer <token>` 헤더로 붙입니다. 이 재사용을 위해 HTTP 호출을 감싸는 얇은 `ApiClient` 헬퍼를 `lib/core/api_client.dart`에 분리해 둡니다. 본체는 이번 편 스펙에 포함되지 않으므로 헬퍼 구현 코드는 생략합니다.

## `.env` 파일과 위험 장면

이 섹션은 `.env` 파일이 실수로 저장소에 올라갈 수 있는 시나리오와 그 위험성을 다룹니다.

JWT 서명 시크릿과 Postgres 비밀번호는 리포지토리에 평문으로 두면 안 됩니다. 이 때문에 레포 루트에 `.env` 파일을 만들고, Dart 서버와 `docker-compose.yml`이 이 파일을 읽도록 고칩니다.

```
# .env  (gitignore됨)
JWT_SECRET=replace-me-with-32-bytes-of-entropy
POSTGRES_PASSWORD=workshop
```

`.gitignore`에 `.env`를 추가합니다. 여기까지는 모든 프로젝트가 하는 일입니다. 문제는 여기서 끝나지 않았습니다. 다음은 이 원고 리허설 중 저자가 만난 장면입니다.

저는 로그인 코드를 다 붙이고 "변경된 파일들을 커밋해줘"라고 Claude Code에게 말했습니다. Claude Code는 `git status`를 먼저 확인했습니다. 출력에 `.env.local`이라는 파일이 눈에 띄었습니다. `.gitignore`에는 `.env`만 있었고, 변종은 빠져 있었습니다. 다음 턴에서 Claude Code가 `git add .`을 호출했습니다. `.env.local`이 스테이지에 올라갔습니다. 다행히 커밋 전에 `git diff --staged`를 확인해 JWT 시크릿이 포함된 것을 발견했지만, 그 한 줄의 확인을 놓쳤다면 원격으로 밀려나갈 뻔했습니다.

한 번 일어나면 되돌리기가 까다롭습니다. 이미 커밋에 포함된 시크릿은 히스토리에서 지우려면 `git filter-repo`나 BFG 같은 도구가 필요하고, 원격에 이미 푸시됐다면 시크릿 로테이션부터 해야 합니다. 이 때문에 이 실수는 "발생한 뒤 수습"이 아니라 "발생 자체를 차단"해야 합니다.

## PreToolUse 훅으로 차단하기

이 섹션은 Claude Code의 훅 기제를 이용해, `git add` 호출이 `.env`류 파일을 포함하면 Bash 도구 실행 자체를 막는 스크립트를 작성합니다.

훅 파일은 셸 스크립트 한 개입니다. 경로는 프로젝트 레벨 `.claude/hooks/deny-env-in-git-add.sh`로 둡니다.

```bash
#!/usr/bin/env bash
# .claude/hooks/deny-env-in-git-add.sh
# PreToolUse 훅: Bash 도구가 git add 관련 명령으로 .env를 끌어들이는 경우 차단
set -euo pipefail

# 훅 입력은 stdin으로 JSON 형태로 들어옵니다.
payload=$(cat)
tool=$(echo "$payload" | jq -r '.tool_name // empty')
cmd=$(echo "$payload" | jq -r '.tool_input.command // empty')

[ "$tool" != "Bash" ] && { echo '{"decision":"approve"}'; exit 0; }

# git add로 .env* 파일이 포함되면 차단합니다. 와일드카드(., -A, -u)는 잠재적 위험으로 간주합니다.
if echo "$cmd" | grep -qE '^git[[:space:]]+add'; then
  if echo "$cmd" | grep -qE '(\.env[^[:space:]]*|[[:space:]]\.[[:space:]]|-A|-u)'; then
    cat <<'JSON'
{"decision":"block","reason":".env*가 스테이징될 위험이 있는 git add 명령입니다. 파일을 명시적으로 지정해주세요."}
JSON
    exit 0
  fi
fi
echo '{"decision":"approve"}'
```

스크립트 파일에 실행 권한을 줍니다(`chmod +x`). 그리고 이 훅이 언제 어떤 도구 호출에 개입하는지를 `settings.json`에 등록합니다.

```jsonc
// .claude/settings.json (프로젝트 레벨)
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/deny-env-in-git-add.sh"
          }
        ]
      }
    ]
  }
}
```

이제 Claude Code 세션에서 `git add .`나 `git add -A`를 호출하려 하면, 훅이 먼저 개입해 Bash 실행을 차단합니다. 차단 사유는 `reason` 필드로 모델에게 전달되며, 모델은 이를 보고 "파일을 명시적으로 지정해야 한다"를 인식합니다. Done When ②가 여기서 채워집니다.

## 왜 프로젝트 레벨에 두었는가 — 설정 계층

훅 등록 위치를 `.claude/settings.json` (프로젝트 레벨)으로 선택한 이유를 이 섹션에서 다룹니다. 이는 "설정 계층"이라는 하네스 부품의 구조에서 나오는 판단입니다.

Claude Code는 `settings.json`을 세 층으로 병합합니다.

- **user 층**: `~/.claude/settings.json`. 개인 전역 설정. 이 프로젝트가 아닌 다른 프로젝트에서도 상속됩니다.
- **project 층**: 프로젝트 루트의 `.claude/settings.json`. 이 프로젝트를 체크아웃한 모든 개발자가 공유합니다.
- **local 층**: 같은 폴더의 `.claude/settings.local.json`. **gitignore됩니다**. 개별 개발자 혹은 특정 머신 전용.

`.env` 차단 훅은 "이 프로젝트의 모든 개발자가 동일한 안전망을 공유해야 한다"는 속성 때문에 project 층에 속합니다. 만약 local 층에 두면, 팀에 새로 합류한 개발자의 머신에는 훅이 없어 같은 사고가 재발할 수 있습니다. 반대로 user 층에 두면, 이 저자의 다른 프로젝트(예: `.env`를 쓰지 않는 문서 전용 레포)에서 불필요하게 개입합니다. project 층이 유일하게 맞는 자리입니다.

그렇다면 local 층은 무엇에 쓰는가. API 키, 개인 토큰, 실험용 MCP 서버 자격 증명 — 즉 **절대 공유되면 안 되는 키 값**이 여기 들어갑니다. 이러한 키를 커밋할 위험은 "훅이 있으면 자동 차단된다"보다 "파일 자체가 gitignore되어 있다"로 먼저 막는 편이 안전합니다. `.gitignore`에 `.claude/settings.local.json`이 들어있는지 확인하고, 들어있지 않다면 이번 편에서 추가합니다.

## 권한 모드 네 가지

훅은 "어떤 조건이면 차단한다"를 결정합니다. 그 앞 단계에서 "어떤 도구 호출에 사용자 확인을 받을 것인가"는 **권한 모드**가 결정합니다. 권한 모드는 네 가지입니다.

- **default**: 파일 쓰기, Bash 실행 등 "부작용이 있는" 도구는 매번 사용자 확인을 받습니다. 가장 보수적입니다.
- **acceptEdits**: 파일 편집(Edit/Write)은 사용자 확인 없이 허용하지만, Bash 실행과 네트워크 접근 계열 도구는 여전히 확인을 받습니다. 작업 집중도가 높을 때 유용합니다.
- **plan**: 읽기 전용. 파일 수정, Bash 실행이 모두 차단됩니다. 설계를 먼저 합의한 뒤 구현으로 넘어가는 용도입니다. P5에서 자세히 다룹니다.
- **bypassPermissions**: 모든 도구 호출을 확인 없이 허용합니다. 자동화 파이프라인 같은 제어된 환경에서만 적절합니다.

권한 모드는 훅과 독립적으로 동작합니다. 예를 들어 `bypassPermissions` 모드에서도 PreToolUse 훅이 `.env` 포함 `git add`를 차단합니다. 훅은 "사용자가 실수로 관대한 권한 모드를 선택한 순간에도 동작하는 안전망"으로 기능합니다. 이 독립성이 훅이 권한 모드보다 하위 층에 있다는 증거입니다.

## 해부: 훅·설정 계층·권한 모드가 맞물리는 방식

이 편에서 해부하는 세 개념은 독립 정의가 아니라 순서를 가진 체인입니다.

> **해부: 도구 호출 주변의 세 층**
>
> Claude Code가 도구를 호출하는 순간, 프롬프트에서 도구 실제 실행까지 사이에 세 개의 게이트가 있습니다.
>
> **① 권한 모드**가 가장 먼저 개입합니다. `plan` 모드라면 쓰기·Bash가 아예 차단되어 훅도 실행되지 않습니다. `default` 모드라면 사용자에게 확인 프롬프트를 띄웁니다. `bypassPermissions`라면 바로 통과합니다.
>
> **② PreToolUse 훅**이 그 다음입니다. 권한 모드가 통과된 도구 호출에 대해, 훅이 최종 판단을 내립니다. 훅의 `decision` 필드가 `block`이면 도구 실행은 일어나지 않고, 그 사유는 모델의 대화 컨텍스트로 되돌아갑니다. `approve`면 실행이 이어집니다. 훅의 `matcher` 필드는 "이 훅이 어떤 도구 호출에만 개입할지"를 좁힙니다. `"Bash"`는 Bash 도구에만, `"Edit|Write"`는 파일 쓰기 도구에만, `"*"`은 모든 도구 호출에 반응합니다.
>
> **③ PostToolUse 훅**은 실행 뒤에 붙습니다. 실행 결과를 보고 로그를 남기거나, 실패 시 후속 액션을 트리거합니다. 이 시리즈에서는 P6에서 PostToolUse를 활용합니다.
>
> 설정 계층은 이 세 게이트의 **위치**를 결정합니다. 권한 모드의 기본값, 훅 등록, 도구 허용 목록은 모두 `settings.json`에 정의됩니다. user → project → local 순서로 병합되며, 같은 키가 겹치면 **뒤쪽 층이 이깁니다**(local이 project를, project가 user를 덮어씁니다). 팀 공유 규칙은 project에, 개인 머신 전용 규칙은 local에 두는 것이 자연스러운 이유입니다.
>
> 또 하나의 이벤트가 있습니다. **SessionStart 훅**은 도구 호출과 무관하게, 새 세션이 열릴 때마다 실행됩니다. 환경 변수 로딩, 현재 브랜치 출력, 최근 작업 메모 주입 같은 세션 컨텍스트 준비에 씁니다. gstack의 프리앰블이 대표적인 SessionStart 소비자입니다.

이 세 게이트 중 어느 하나에 자신의 안전망을 놓아야 할지는 안전망의 성격이 결정합니다. "특정 명령을 본질적으로 금지해야 한다"면 훅입니다. "이 프로젝트 전체가 신중 모드여야 한다"면 권한 모드의 기본값입니다. "이 머신에서만 허용할 실험적 설정"이라면 local 층입니다.

## Done When

- [ ] `curl -X POST localhost:8080/login -d '{"email":"a@b.com","password":"x"}'`이 유효한 자격 증명에 대해 200 + JWT를 반환하고, 만료된 JWT로 보호된 엔드포인트를 호출하면 401을 반환합니다.
- [ ] `.claude/hooks/deny-env-in-git-add.sh` 훅이 등록된 상태에서 `git add .` 또는 `git add -A`를 시도하면 `decision: block`으로 실행이 멈춘다는 점을 `echo '{"tool_name":"Bash","tool_input":{"command":"git add ."}}' | .claude/hooks/deny-env-in-git-add.sh` 로 테스트해 확인할 수 있습니다.
- [ ] `.claude/settings.local.json`이 `.gitignore`에 포함되어 있고, `git check-ignore -v .claude/settings.local.json`이 대응 규칙을 출력합니다.

## 마치며

이번 편에서는 로그인 E2E를 붙이고, `.env` 파일이 실수로 스테이징되는 순간을 PreToolUse 훅 하나로 차단했습니다. 그리고 훅을 어느 층에 등록할지 결정하기 위해 `settings.json`의 세 층 구조를 뜯어보고, 훅의 개입 앞 단계에 있는 권한 모드 네 가지를 정리했습니다.

이 편의 요지는 "훅을 쓰라"가 아닙니다. 요지는 "도구 호출 하나가 실제로 실행되기까지 세 개의 게이트를 지난다"는 구조 인식입니다. 이러한 구조를 머리에 두면, 안전망을 어디에 놓을지 — 권한 모드 / 훅 / 설정 층 — 를 성격별로 판단할 수 있게 됩니다. 실수를 막는 방식은 하나가 아닙니다. 하네스는 서로 다른 타이밍에 서로 다른 층이 개입하도록 설계되어 있고, 이 편은 그 구조를 처음으로 직접 만져본 편입니다.

다음 편 v0.4에서는 스레드 CRUD를 붙입니다. 그 과정에서 처음으로 서브에이전트 두 개를 병렬로 띄워, UI와 API를 동시에 구현합니다. 단일 에이전트와 서브에이전트의 컨텍스트 격리가 어떻게 갈라지는지, 그리고 두 결과를 병합할 때 스키마 이름 충돌이 왜 생기는지를 실제로 경험합니다.

## 이 편 끝난 상태

v0.3 태그가 가리키는 상태입니다.

- `apps/dart_server/`: `/login` 엔드포인트 추가, JWT 만료 시간 설정, `verify` 헬퍼
- `apps/flutter_client/`: 로그인 화면 + JWT `shared_preferences` 저장
- 레포 루트 `.env` (gitignore됨) + `.env.example`
- `.claude/hooks/deny-env-in-git-add.sh` (실행 권한 부여됨)
- `.claude/settings.json`: PreToolUse 훅 등록
- `.claude/settings.local.json` gitignored 확인 완료
- `.gitignore`에 `.env*`, `.claude/settings.local.json` 포함

## 참조

- dart_jsonwebtoken 검증 API: https://pub.dev/packages/dart_jsonwebtoken
- shared_preferences: https://pub.dev/packages/shared_preferences
- Claude Code 훅 공식 문서: https://docs.claude.com/en/docs/claude-code/hooks
- settings.json 레퍼런스: https://docs.claude.com/en/docs/claude-code/settings
- 권한 모드 개요: https://docs.claude.com/en/docs/claude-code/permissions
