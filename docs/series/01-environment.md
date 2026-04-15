# P1 — 환경 세팅: 스택과 하네스 인스턴스 준비

## TL;DR

- 이번 편에서 만드는 것: Flutter 클라이언트 + Dart 서버 + PostgreSQL 컨테이너 세 개가 각자 hello world를 찍는 상태.
- 이번 편에서 해부하는 하네스 개념: **스킬이 로드되는 순간**. 슬래시 명령을 쳤을 때 Claude Code 안쪽에서 어떤 파일이 언제 프롬프트에 주입되는지를 봅니다.
- 이 편 끝나면 `docker-compose up`, `curl localhost:8080/ping`, `flutter run` 세 명령이 모두 성공합니다.

## 지난 편 요약

v0.0에서는 레포 골격, 설계서, 용어 pin, 버전 pin을 확정하고 `v0.0` 태그를 달았습니다. 이번 편부터는 실제 코드가 쌓이기 시작합니다.

## 이번 편 목표

하네스를 해부하려면 해부 대상이 움직이고 있어야 합니다. 이를 위해 MVP가 돌아갈 가장 얇은 뼈대 — 서버가 뜨고, 클라이언트가 그 서버에 접속하고, DB가 기동되어 있는 상태 — 를 먼저 만듭니다. 기능은 없습니다. 세 컨테이너가 서로 hello world만 주고받습니다.

같이 하는 일 한 가지가 더 있습니다. 이 상태를 만드는 동안 처음으로 Claude Code 슬래시 명령을 하나 실행하고, 그 순간 하네스 내부에서 벌어지는 일을 해부합니다.

## 레이아웃 결정

이 섹션은 레포 내부 디렉토리 배치를 먼저 정리합니다. 이후 편들이 이 구조를 전제로 쓰이기 때문입니다.

```
.
├── apps/
│   ├── dart_server/        # shelf 기반 HTTP 서버
│   └── flutter_client/     # Flutter 앱
├── db/
│   └── init.sql            # Postgres 초기화
├── docker-compose.yml
└── docs/
```

모노레포로 가는 이유는 독자가 `git clone` 한 번으로 글과 코드를 모두 확보할 수 있기 때문입니다. 또한 태그 네임스페이스가 루트에 단일하게 존재해, 특정 편의 상태를 `git checkout v0.N`으로 반복 가능하게 재현할 수 있습니다.

## Docker Compose로 PostgreSQL 띄우기

먼저 `docker-compose.yml`을 작성해 Postgres 16을 로컬에 기동합니다.

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: workshop
      POSTGRES_PASSWORD: workshop
      POSTGRES_DB: workshop
    ports:
      - "5432:5432"
    volumes:
      - ./.postgres-data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
```

`POSTGRES_PASSWORD`를 평문으로 적는 것이 거슬린다면 올바른 감각입니다. P3에서 `.env` 파일로 분리하며 훅으로 방어합니다. 지금은 의도적으로 평문으로 두고, 이후 편에서 이 선택이 왜 문제인지를 재구성합니다.

`db/init.sql`은 다음 한 줄만 담습니다.

```sql
-- db/init.sql
SELECT 'postgres is up' AS status;
```

`docker-compose up -d`를 실행하면 Postgres가 올라옵니다.

## Dart 서버 부트스트랩

다음은 shelf 기반의 최소 HTTP 서버입니다. `/ping` 엔드포인트 하나만 갖습니다.

`apps/dart_server/pubspec.yaml`의 의존성은 다음과 같이 정확 버전으로 고정합니다.

```yaml
# apps/dart_server/pubspec.yaml
name: dart_server
environment:
  sdk: "^3.4.0"

dependencies:
  shelf: 1.4.1
  shelf_router: 1.1.4
```

`^` 범위 연산자를 쓰지 않는 이유는 시리즈 전체의 재현성 때문입니다. 이 시리즈는 1년 뒤에도 `git checkout v0.1 && dart pub get && dart run`으로 같은 상태가 재현되어야 합니다. 이 때문에 버전 범위를 받지 않고, `pubspec.lock`까지 커밋 대상에 포함합니다.

한 가지 예외는 `environment.sdk`의 `^3.4.0`입니다. 이 자리의 `^`는 하한 호환 SDK를 선언하는 관례이며, `dart pub get`이 런타임의 Dart 버전을 고정하는 수단이 아닙니다. 패키지 버전 고정과는 성격이 다르기 때문에 그대로 둡니다. 또한 P1 시점에서는 `shelf`와 `shelf_router`만 pin합니다. P2 이후 `postgres`, `bcrypt`, `dart_jsonwebtoken`이 도입되는 편에서 각각 해당 편의 `pubspec.yaml`에 정확 버전으로 추가됩니다. 쓰지도 않는 의존성을 미리 박아두면 `dart pub get`이 불필요한 패키지를 내려받고 lock이 커집니다. 이 시리즈는 편마다 "그 편에서 실제로 쓰는 것만 pin"을 원칙으로 삼습니다.

서버 본체는 다음과 같습니다.

```dart
// apps/dart_server/bin/server.dart
import 'package:shelf/shelf.dart';
import 'package:shelf/shelf_io.dart' as io;
import 'package:shelf_router/shelf_router.dart';

void main() async {
  final router = Router()
    ..get('/ping', (Request req) => Response.ok('pong'));

  final handler = Pipeline()
      .addMiddleware(logRequests())
      .addHandler(router.call);

  await io.serve(handler, '0.0.0.0', 8080);
  print('dart_server listening on :8080');
}
```

`dart run bin/server.dart`로 실행한 뒤 `curl localhost:8080/ping`을 호출하면 `pong`이 돌아옵니다. 서버가 로그로 요청을 찍고 있다면, 첫 번째 뼈대가 완성된 것입니다.

## Flutter 클라이언트 부트스트랩

Flutter 앱은 `flutter create` 명령으로 생성하되, 생성 직후 `pubspec.yaml`의 핵심 패키지를 정확 버전으로 고정합니다.

```bash
% cd apps
% flutter create flutter_client
% cd flutter_client
```

명령이 완료되면 `apps/flutter_client/` 디렉토리가 생성되고 기본 카운터 앱 템플릿이 들어 있습니다. 이 템플릿을 그대로 두지 않고, `pubspec.yaml`에서 서버 통신에 필요한 패키지만 정확 버전으로 pin합니다.

```yaml
# apps/flutter_client/pubspec.yaml (발췌)
name: flutter_client
environment:
  sdk: "^3.4.0"
  flutter: "3.22.0"

dependencies:
  flutter:
    sdk: flutter
  http: 1.2.2
  shared_preferences: 2.3.2
```

서버 측과 동일한 원칙입니다. 쓰지도 않을 패키지를 미리 박아두지 않습니다. `provider`나 상태관리 라이브러리는 실제로 도입되는 편에서 pin합니다. `flutter pub get`을 실행한 뒤 `pubspec.lock`을 커밋 대상에 포함시키면, 이 편의 재현 조건이 완성됩니다.

`lib/main.dart`는 서버의 `/ping`을 호출해 응답을 화면에 표시합니다.

```dart
// apps/flutter_client/lib/main.dart
import 'package:flutter/material.dart';
import 'package:http/http.dart' as http;

void main() => runApp(const App());

class App extends StatelessWidget {
  const App({super.key});
  @override
  Widget build(BuildContext context) => const MaterialApp(home: PingScreen());
}

class PingScreen extends StatefulWidget {
  const PingScreen({super.key});
  @override
  State<PingScreen> createState() => _PingScreenState();
}

class _PingScreenState extends State<PingScreen> {
  String _status = 'tap to ping';

  Future<void> _ping() async {
    final res = await http.get(Uri.parse('http://localhost:8080/ping'));
    setState(() => _status = res.body);
  }

  @override
  Widget build(BuildContext context) => Scaffold(
    body: Center(child: Text(_status)),
    floatingActionButton: FloatingActionButton(onPressed: _ping, child: const Icon(Icons.wifi)),
  );
}
```

시뮬레이터에서 `flutter run`을 실행하고 버튼을 누르면 화면이 `pong`으로 바뀝니다. 단, iOS 시뮬레이터에서 `localhost`가 호스트 머신을 가리키지 않는 경우가 있습니다. 이때는 `localhost`를 호스트 IP로 바꿔주거나 Android 에뮬레이터의 `10.0.2.2`를 씁니다.

## 하네스 인스턴스 버전 고정

이 섹션은 본 시리즈가 해부 표본으로 고른 하네스 인스턴스의 버전을 확인하는 절차입니다. 표본을 고정하지 않으면 이후 편의 해부 결과가 독자의 환경에서 다르게 재현될 여지가 생기기 때문입니다.

표본 스킬·훅 묶음 층으로 고른 것은 gstack이고, P0에서 선언한 버전은 0.16.4.0입니다. 현재 설치된 버전을 확인합니다.

```bash
% cat ~/.claude/skills/gstack/VERSION
0.16.4.0
```

이 출력이 `0.16.4.0`이면 표본 버전이 맞춰진 상태입니다. 버전이 다르면 이후 편에서 예시로 나오는 슬래시 명령의 이름이나 프리앰블 출력 형태가 달라질 수 있습니다. 다른 스킬·훅 묶음 층(예: 다른 프리셋)을 쓰는 독자라면, 이후 편의 해부 결과를 자신의 인스턴스 용어로 치환해서 읽으면 됩니다. 층 구성과 끼어드는 순서는 대체로 같습니다.

> **사이드바: `/office-hours`는 이 시리즈의 기원입니다**
>
> 시리즈 설계 자체가 `/office-hours` 스킬을 통해 만들어졌습니다. 설계 문서 `/docs/series/DESIGN.md`는 그 결과물입니다. 다만 이 시리즈 본문에서 `/office-hours`를 깊게 다루지는 않습니다. 이 스킬에 대한 기록은 각주 수준으로 언급하고, 본 해부는 스킬·훅·서브에이전트·권한 모드에 집중합니다.

## 첫 슬래시 명령, 그리고 그 순간 내부에서 벌어지는 일

이제 Claude Code를 열고, 아주 가벼운 스킬 하나를 실행해봅니다. 예를 들어 `/health`를 쳐서 현재 프로젝트의 상태를 확인해봅니다.

```
> /health
```

결과가 돌아오기까지는 수 초입니다. 그 수 초 동안 눈에 보이지 않는 일이 여러 단계로 일어납니다. 이 시리즈의 첫 해부 대상이 바로 이 순간입니다.

> **해부: 스킬이 로드되는 순간**
>
> 슬래시 명령이 입력되면 Claude Code는 해당 이름의 스킬을 디스크에서 찾습니다. 탐색 경로는 사용자 레벨(`~/.claude/skills/`), 프로젝트 레벨(`.claude/skills/`) 순서입니다. 일치하는 `SKILL.md`를 찾으면, 그 파일의 전체 내용이 모델 프롬프트에 주입됩니다.
>
> `SKILL.md` 최상단의 **프런트매터(frontmatter)** — `description`, `name` 같은 YAML 블록 — 는 주입 전 단계에서 스킬의 메타데이터로 읽힙니다. `description`은 다른 문맥에서 자동 트리거 여부를 판별할 때 쓰입니다.
>
> 본문에 포함된 **프리앰블** bash 블록은 스킬 시작 시점에 실행되어 업데이트 체크, 세션 카운트, 분석 로그 같은 부가 작업을 수행합니다. 이 블록의 출력은 모델의 대화 컨텍스트 앞단에 그대로 주입됩니다. 이에 따라 프리앰블에 찍힌 값(버전, 브랜치, 학습 로그 엔트리 수 등)은 모델이 이 스킬 실행의 첫 프롬프트로 보게 됩니다.
>
> 정리하면 `슬래시 명령 → SKILL.md 탐색 → 프런트매터 파싱 → 프리앰블 실행 → 출력이 프롬프트에 주입 → 본문 지시가 모델 컨텍스트에 들어감`의 순서입니다. 이 순서를 기억해두면, 다음 편에서 직접 스킬을 작성할 때 각 부분이 어디에 어떻게 걸리는지가 분명해집니다.

이 해부 하나를 머리에 두고 다음 편으로 넘어갑니다.

## Done When

- [ ] `docker-compose up -d`로 Postgres 컨테이너가 기동되고, `docker ps`에 `postgres:16`이 보입니다.
- [ ] `curl localhost:8080/ping`이 `pong`을 200으로 반환합니다.
- [ ] Flutter 앱이 시뮬레이터 또는 에뮬레이터에서 빌드되고, 버튼을 누르면 화면이 `pong`으로 바뀝니다.
- [ ] `apps/dart_server/pubspec.yaml`과 `apps/flutter_client/pubspec.yaml` 양쪽에서 주요 패키지가 정확 버전(`^` 없음)으로 고정되어 있고, 두 `pubspec.lock`이 모두 커밋되어 있습니다.

## 원고 작성 기법 고지

이 편은 실제 작성에 앞서 부트스트랩을 한 번 리허설한 뒤에 재구성된 글입니다. "지금 처음 해본 것처럼" 읽히지만, 실제 경로는 한 번 걸어본 길입니다. 재연의 목적은 독자가 처음 따라갈 때 막힐 지점을 미리 걷어내기 위함입니다.

## 마치며

이번 편에서는 세 컨테이너가 서로 hello world를 주고받는 상태까지 도달했고, 첫 슬래시 명령을 통해 스킬이 로드되는 순간을 해부했습니다.

이 상태가 의미하는 것은 단순히 "환경이 세팅되었다"가 아닙니다. 이제부터의 모든 편은 이 뼈대 위에 하네스 부품 하나씩을 올려가며, 각 부품이 어느 순간 어디에 개입하는지를 드러내는 여정입니다. 이러한 맥락에서 이 편의 진짜 산출물은 돌아가는 세 컨테이너가 아니라, "슬래시 명령 하나에도 내부에 순서가 있다"는 관점 자체입니다.

다음 편 v0.2에서는 회원가입 기능을 만들며, 작업 중 반복 패턴이 보이는 순간 커스텀 스킬을 하나 직접 작성합니다. 그 과정에서 `SKILL.md`의 내부 구조를 처음으로 손수 써보게 됩니다.

## 이 편 끝난 상태

v0.1 태그가 가리키는 상태입니다.

- `docker-compose.yml`, `db/init.sql`
- `apps/dart_server/`: `/ping` 엔드포인트만 있는 shelf 서버, `pubspec.yaml` + `pubspec.lock`
- `apps/flutter_client/`: `/ping`을 호출해 화면에 표시하는 Flutter 앱, `pubspec.yaml` + `pubspec.lock`
- 표본 하네스 스킬·훅 묶음 층: gstack 0.16.4.0 설치 확인됨

## 참조

- shelf 패키지: https://pub.dev/packages/shelf
- shelf_router: https://pub.dev/packages/shelf_router
- Flutter 설치: https://docs.flutter.dev/get-started/install
- Docker Compose 문서: https://docs.docker.com/compose/
