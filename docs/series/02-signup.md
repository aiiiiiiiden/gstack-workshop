# P2 — 회원가입: 반복에서 커스텀 스킬로

## TL;DR

- 이번 편에서 만드는 것: `POST /signup` 엔드포인트, `users` 테이블, Flutter 회원가입 화면. 그리고 `.claude/skills/scaffold-feature/SKILL.md` 하나.
- 이번 편에서 해부하는 하네스 개념: **스킬 파일 하나의 구조**. 프런트매터, `description`의 자동 트리거 역할, 재귀 금지 규칙, 디렉토리 관례까지.
- 이 편 끝나면 회원가입이 E2E로 돌아가고, 동일한 "DB 테이블 + Dart 모델 + Flutter 모델" 묶음을 한 줄 슬래시 명령으로 뼈대 잡을 수 있습니다.

> **재연 고지**: 이 편 후반부의 `bookmarks` 기능 데모는 스킬 동작을 확인하기 위한 일회성 실험이며, `v0.2` 태그에는 남아 있지 않습니다. 파일을 되돌린 뒤 이 글을 작성했습니다. 재연임을 인지하고 읽어주시기 바랍니다.

## 지난 편 요약

v0.1에서는 Postgres 컨테이너, Dart 서버, Flutter 클라이언트 세 개가 hello world를 주고받는 뼈대를 만들고 `v0.1` 태그를 달았습니다. 또한 첫 슬래시 명령을 실행하면서, 스킬이 로드되는 순간(`SKILL.md` 탐색 → 프런트매터 파싱 → 프리앰블 실행 → 프롬프트 주입)의 내부 순서를 해부했습니다.

이번 편에서는 그 뼈대 위에 첫 기능을 올립니다. 그리고 기능을 만드는 과정 자체에서, 스킬이라는 기제가 왜 존재하는지를 손으로 느낍니다.

## 이번 편 목표

회원가입 한 기능을 E2E로 붙이는 것이 겉 목표입니다. 속 목표는 다릅니다. 이 편은 "같은 모양의 코드를 세 번 치는 순간"을 의도적으로 연출해, 그 순간에 커스텀 스킬 하나가 어떻게 반복을 흡수하는지를 보입니다. 스킬은 거창한 자동화 도구가 아닙니다. 가장 얇게 쓰일 때는, 이미 손으로 세 번 한 일을 한 줄로 재현하는 메모에 가깝습니다.

## users 테이블과 DB 마이그레이션

먼저 Postgres에 `users` 테이블을 준비합니다. 이 시리즈는 마이그레이션 도구를 따로 쓰지 않습니다. `db/migrations/` 아래에 순번 붙은 SQL 파일을 두고, 서버 기동 시점에 순서대로 실행하는 방식으로 단순화합니다.

```sql
-- db/migrations/001_users.sql
CREATE TABLE IF NOT EXISTS users (
  id         BIGSERIAL PRIMARY KEY,
  email      TEXT NOT NULL UNIQUE,
  password   TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

`password` 컬럼에 들어갈 값은 평문이 아닌 bcrypt 해시입니다. 이 규약을 컬럼 이름으로 드러내지 않은 이유는, 이후 편에서 패스워드 정책이 바뀌어도 컬럼 이름을 바꾸지 않게 하기 위함입니다. 대신 Dart 모델에 주석으로 고정합니다.

## Dart 서버 — /signup 엔드포인트

이 섹션은 회원가입 요청을 받아 bcrypt로 해시하고 JWT를 발급해 돌려주는 핸들러를 구현합니다. 서버 측 Done When ①과 ②가 이 섹션에서 채워집니다.

서버 의존성에 `postgres`와 `bcrypt`, `dart_jsonwebtoken`을 정확 버전으로 추가합니다.

```yaml
# apps/dart_server/pubspec.yaml (이번 편 추가분)
dependencies:
  shelf: 1.4.1
  shelf_router: 1.1.4
  postgres: 3.2.0
  bcrypt: 1.1.3
  dart_jsonwebtoken: 2.14.0
```

이전 편과 동일한 원칙입니다. `^` 범위 연산자는 쓰지 않고, `pubspec.lock`을 그대로 커밋합니다.

다음은 `POST /signup` 핸들러입니다. 본문은 JSON으로 `email`과 `password`를 받습니다. 응답은 201과 함께 JWT를 돌려줍니다.

```dart
// apps/dart_server/bin/server.dart (발췌)
Future<Response> _signup(Request req) async {
  final body = jsonDecode(await req.readAsString()) as Map<String, dynamic>;
  final email = (body['email'] as String).trim().toLowerCase();
  final password = body['password'] as String;

  final hash = BCrypt.hashpw(password, BCrypt.gensalt());
  final row = await db.execute(
    Sql.named('INSERT INTO users(email, password) VALUES(@e, @p) RETURNING id'),
    parameters: {'e': email, 'p': hash},
  );
  final id = row.first[0] as int;

  final jwt = JWT({'sub': id, 'email': email}).sign(SecretKey(_jwtSecret));
  return Response(201,
    body: jsonEncode({'id': id, 'token': jwt}),
    headers: {'content-type': 'application/json'},
  );
}
```

`curl` 한 줄로 검증합니다. `{"email":"a@b.com","password":"hunter2"}`를 POST하면 `201`이 돌아오고, 응답 JSON에 `token` 필드가 포함됩니다. `psql`로 접속해 `SELECT id, email FROM users`를 던지면 새 행이 한 개 들어있습니다. 이것이 서버 측 Done When ①과 ②입니다.

## Flutter 클라이언트 — 회원가입 화면

Flutter 쪽은 form + validation + http 호출을 얹습니다. 상태관리 라이브러리는 아직 도입하지 않습니다. `StatefulWidget`의 기본 `setState`로 충분합니다. 유효성 검사는 `Form` + `GlobalKey<FormState>` + `TextFormField.validator` 조합으로 클라이언트 단에서 우선 걸러냅니다. 빈 값이나 4자 미만 비밀번호는 서버까지 가지 않습니다.

```dart
// apps/flutter_client/lib/features/signup/signup_screen.dart
class SignupScreen extends StatefulWidget {
  const SignupScreen({super.key});
  @override
  State<SignupScreen> createState() => _SignupScreenState();
}

class _SignupScreenState extends State<SignupScreen> {
  final _formKey = GlobalKey<FormState>();
  final _email = TextEditingController();
  final _password = TextEditingController();
  String _status = '';

  Future<void> _submit() async {
    if (!_formKey.currentState!.validate()) return;
    final res = await http.post(
      Uri.parse('$kApiBase/signup'),
      headers: {'content-type': 'application/json'},
      body: jsonEncode({'email': _email.text.trim(), 'password': _password.text}),
    );
    setState(() => _status = res.statusCode == 201 ? 'ok' : 'fail ${res.statusCode}');
  }

  @override
  Widget build(BuildContext context) => Scaffold(
    appBar: AppBar(title: const Text('Signup')),
    body: Padding(
      padding: const EdgeInsets.all(16),
      child: Form(
        key: _formKey,
        child: Column(children: [
          TextFormField(
            controller: _email,
            decoration: const InputDecoration(labelText: 'email'),
            validator: (v) => (v == null || !v.contains('@')) ? 'invalid email' : null,
          ),
          TextFormField(
            controller: _password,
            decoration: const InputDecoration(labelText: 'password'),
            obscureText: true,
            validator: (v) => (v == null || v.length < 4) ? 'min 4 chars' : null,
          ),
          ElevatedButton(onPressed: _submit, child: const Text('submit')),
          Text(_status),
        ]),
      ),
    ),
  );
}
```

시뮬레이터에서 빈 값으로 제출하면 두 필드 아래에 오류 메시지가 뜨고 네트워크 호출은 발생하지 않습니다. 제대로 된 값을 넣고 제출하면 화면 하단에 `ok`가 찍힙니다. 이로써 클라이언트 측 Done When ③의 절반이 채워집니다. 나머지 절반은 다음 섹션에서 채워집니다.

## 반복이 보이기 시작합니다

회원가입 한 기능을 위에 붙이고 나면, 파일 세 개가 같은 "모양"을 하고 있음이 드러납니다.

- `db/migrations/001_users.sql`: `users` 테이블 정의
- `apps/dart_server/lib/models/user.dart`: 서버 쪽 User 클래스
- `apps/flutter_client/lib/models/user.dart`: 클라이언트 쪽 User 클래스

세 파일의 필드 이름, 타입, nullable 여부가 거의 동일합니다. 차이는 언어 문법과 `snake_case` ↔ `camelCase` 변환뿐입니다. 이 상태에서 다음 기능(스레드, 댓글, 좋아요)을 추가한다면 같은 작업을 세 번 더 하게 됩니다.

이 시점에서 두 가지 선택지가 있습니다. 하나는 "그냥 손으로 계속 친다." 다른 하나는 "이 패턴을 스킬로 뽑아서, 다음부터는 한 줄로 뼈대를 만든다." 이 시리즈의 주제가 하네스이므로, 두 번째 길을 갑니다. 이 때문에 다음 섹션은 첫 커스텀 스킬 작성입니다.

## /scaffold-feature 스킬 작성

스킬은 프로젝트 레벨(`.claude/skills/`)과 사용자 레벨(`~/.claude/skills/`) 두 군데에 둘 수 있습니다. 이번 스킬은 이 프로젝트 전용이므로 프로젝트 레벨을 선택합니다. 디렉토리 이름이 곧 스킬 이름이며, 슬래시 명령으로 호출될 때 그대로 사용됩니다.

```
.claude/
└── skills/
    └── scaffold-feature/
        └── SKILL.md
```

`SKILL.md`의 전체 내용은 다음과 같습니다. 최상단의 YAML 블록이 프런트매터입니다. 본문은 마크다운입니다.

```markdown
---
name: scaffold-feature
description: Scaffold a new feature's DB migration, Dart server model, and Flutter client model in one pass. Use when the user asks to add a new feature with a backing table (e.g., "add threads feature", "create comments").
---

# /scaffold-feature

새 기능 하나를 위한 세 파일의 뼈대를 동시에 생성합니다.

## 입력

사용자로부터 다음을 받습니다.
1. 기능 이름 (예: `threads`, `comments`) — 복수형, snake_case.
2. 필드 목록 — 각 필드의 이름과 타입, nullable 여부.

## 출력

다음 세 파일을 작성합니다. 이미 존재하면 덮지 않고 사용자에게 확인합니다.

- `db/migrations/NNN_<feature>.sql` — `CREATE TABLE <feature>` + 공통 컬럼(`id`, `created_at`).
- `apps/dart_server/lib/models/<feature_singular>.dart` — final 필드를 가진 클래스, `fromRow` / `toJson` 포함.
- `apps/flutter_client/lib/models/<feature_singular>.dart` — `fromJson` 포함.

## 동작 순서

1. 사용자 입력을 파싱한다.
2. 세 파일의 경로를 계산한다. 기존 파일이 있으면 중단하고 사용자에게 알린다.
3. 각 파일을 Write 도구로 생성한다.
4. 생성된 경로 세 개를 출력한다.

## 주의

- 이 스킬은 테이블/모델의 뼈대만 만듭니다. 핸들러(엔드포인트)와 화면은 만들지 않습니다. 이후 작업은 사용자가 이어서 진행합니다.
- 스킬 내부에서 다시 `/scaffold-feature`를 호출하지 않습니다. 슬래시 명령의 재귀 호출은 안전하지 않습니다.
```

파일을 저장한 뒤 새 Claude Code 세션을 열면 `/scaffold-feature`가 슬래시 명령 목록에 등장합니다. 기존 세션에서는 스킬 디스크 스캔이 세션 시작 시에만 일어나므로, 재시작이 필요합니다.

## 스킬 재실행으로 검증

이 스킬이 진짜로 동작하는지를 확인하기 위해, 두 번째 기능의 뼈대를 이 스킬로 잡습니다. 가상의 `bookmarks` 기능을 만들어보는 재연 데모입니다(이 편 상단의 재연 고지 참고).

```
> /scaffold-feature
```

슬래시 명령을 치면 Claude Code가 스킬의 본문 지시를 읽고 대화형으로 입력을 묻습니다. 기능 이름을 `bookmarks`로 주고, 필드로 `user_id: bigint`, `url: text`, `title: text?`를 넣습니다. `?`는 nullable을 의미하며, 스킬 본문의 "입력 스키마"에서 그렇게 약속해둔 기호입니다.

출력은 세 파일의 경로 목록입니다.

```
db/migrations/002_bookmarks.sql
apps/dart_server/lib/models/bookmark.dart
apps/flutter_client/lib/models/bookmark.dart
```

각 파일을 열어보면 users 테이블·모델과 동일한 골격이 들어있습니다. 복수형 `bookmarks` 테이블, 단수형 `Bookmark` 클래스, `fromRow` / `fromJson` / `toJson` 세 메서드. 수동으로 같은 작업을 하면 세 파일을 돌아다니며 타입과 이름을 맞추는 데 최소 3분이 걸립니다. 스킬을 경유하면 30초 안쪽입니다. 차이는 단순 시간이 아닙니다. "세 파일의 모양이 어긋나지 않는다"가 규약 수준으로 보장된다는 점이 더 큽니다.

Done When ③이 채워졌습니다. 데모가 끝난 뒤 `bookmarks` 파일들은 되돌립니다. 이 기능은 시리즈 스펙에 없으며, 스킬이 동작한다는 증거는 `v0.2` 태그에 포함된 `.claude/skills/scaffold-feature/SKILL.md` 자체로 대신합니다.

## 해부: SKILL.md 하나의 구조

이 해부 박스는 `SKILL.md`가 프런트매터와 본문, 그리고 암묵적인 규칙 몇 가지로 구성된다는 사실을 해체합니다.

> **해부: 스킬 파일의 구조**
>
> **프런트매터**는 파일 최상단의 YAML 블록입니다. `name`은 디렉토리 이름과 일치해야 합니다. 진짜 중요한 필드는 `description`입니다. 사용자가 자연어로 "threads 기능 추가해줘"라고 말했을 때, Claude Code가 이 스킬을 자동 트리거할지를 `description`의 키워드로 판단합니다. 잘 쓰인 `description` 한 줄이 슬래시 명령어를 기억하지 못하는 사용자를 스킬에 연결합니다.
>
> **본문**은 마크다운입니다. 스킬이 로드되는 순간 본문 전체가 모델의 대화 컨텍스트에 들어갑니다. 이에 따라 본문은 "사용자에게 보여주는 문서"가 아니라 "모델에게 직접 주는 지시"로 읽어야 합니다. 입력·출력·금지 사항을 명확히 적을수록 모델 동작이 예측 가능해집니다.
>
> **재귀 금지**는 안전 관례입니다. `SKILL.md` 본문 안에서 다시 슬래시 명령을 호출하면 스킬 로드가 중첩되고 컨텍스트가 폭주합니다. 따라서 본문에서는 슬래시 명령을 쓰지 않습니다.
>
> **디렉토리 관례**: 프로젝트 레벨 `.claude/skills/<name>/SKILL.md`, 사용자 레벨 `~/.claude/skills/<name>/SKILL.md`. 두 층에 같은 이름이 있으면 탐색 순서와 무관하게 프로젝트 레벨이 이깁니다. 팀 단위 커스터마이즈가 전역 설정을 안전하게 덮어쓸 수 있다는 뜻입니다.

이 네 가지 — 프런트매터 / 본문 / 재귀 / 디렉토리 — 가 이후 편에서 다루는 훅, 서브에이전트, 권한 모드와 직접 연결됩니다. 다음 편의 훅은 프런트매터가 아니라 `settings.json`에 정의되고, 그 다음 편의 서브에이전트는 별도의 디렉토리 관례를 가집니다.

## Done When

- [ ] `curl -X POST localhost:8080/signup -d '{"email":"a@b.com","password":"x"}'`이 201과 `token` 필드를 포함한 JSON을 반환합니다.
- [ ] `psql`에서 `SELECT id, email FROM users`가 새 행을 보여주고, `password` 컬럼 값이 `$2b$`로 시작하는 bcrypt 해시입니다.
- [ ] `/scaffold-feature`를 호출해 별도 기능 하나의 세 파일을 한 번에 생성할 수 있습니다(`bookmarks` 데모로 검증).

## 마치며

이번 편에서는 회원가입 기능을 E2E로 붙이고, 같은 모양의 반복 코드를 만나는 지점에서 첫 커스텀 스킬 `/scaffold-feature`를 작성했습니다. 그리고 스킬 파일 하나의 구조 — 프런트매터, 본문, 재귀 금지, 디렉토리 관례 — 를 해부했습니다.

이 편의 요지는 "커스텀 스킬을 쓰라"가 아닙니다. 요지는 "손으로 세 번 한 일을 스킬로 뽑는 문턱이 생각보다 낮다"입니다. 이러한 인식이 생기면, 이후 프로젝트에서 반복을 감지한 순간 바로 `SKILL.md` 하나를 까는 반사가 생깁니다. 스킬의 가치는 화려한 자동화가 아니라 이 반사에 있습니다.

다음 편 v0.3에서는 로그인을 붙이며 `.env` 파일을 만듭니다. 그 순간 Claude Code가 `git add .`로 비밀키를 스테이징하려는 장면을 재연하고, PreToolUse 훅으로 그 실수를 차단합니다. 그 과정에서 훅, `settings.json`의 세 층(user / project / local), 권한 모드 네 가지를 한 편에 묶어 해부합니다.

## 이 편 끝난 상태

v0.2 태그가 가리키는 상태입니다.

- `db/migrations/001_users.sql`
- `apps/dart_server/`: `/ping` + `/signup` 두 엔드포인트, `postgres` / `bcrypt` / `dart_jsonwebtoken` 정확 버전 추가, `pubspec.lock` 갱신
- `apps/flutter_client/`: 회원가입 화면 + `User` 모델, `pubspec.lock` 갱신
- `.claude/skills/scaffold-feature/SKILL.md`: 첫 프로젝트 레벨 커스텀 스킬

## 참조

- shelf_router 라우팅 예제: https://pub.dev/packages/shelf_router
- bcrypt 패키지: https://pub.dev/packages/bcrypt
- dart_jsonwebtoken: https://pub.dev/packages/dart_jsonwebtoken
- Claude Code 스킬 개요(공식 문서): https://docs.claude.com/en/docs/claude-code/skills
