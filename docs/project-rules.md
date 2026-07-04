# 필수 프로젝트 규칙

이 에디터에서 작업하는 모든 에이전트와 개발자는 이 규칙을 따른다.

## 기본 규칙

- 의존성은 최신 안정 버전을 채택하되 lockfile에 핀으로 고정한다([tech-stack.md](tech-stack.md)).
- `main`에 직접 푸시하지 않는다. 항상 브랜치와 PR을 사용한다.
- 커밋 메시지는 conventional-commit prefix(`feat`/`fix`/`docs`/`test`/`refactor`/`build`/`ci`/`chore`)를 쓴다. **소문자**로 시작한다.
- 브랜치 이름은 `type/kebab-설명` 형식을 쓴다(예: `feat/live-preview-decorations`).
- 이 프로젝트는 워크스페이스 상위 폴더의 git 리포 하위에 있다. 커밋/PR 범위를 이 프로젝트 폴더로 한정하고, 무관한 상위 폴더 변경을 섞지 않는다.

## 코드 스타일과 린트/포맷

- JS/TS: 린트 `oxlint`, 포맷 `oxfmt`. 커밋 전 통과가 기본이다.
- Rust: 빌드/테스트 `cargo`, 린트 `cargo clippy`(경고를 에러로 취급), 포맷 `cargo fmt`.
- 포맷은 논쟁 대상이 아니다. 포매터 출력에 맞추고 수동 정렬로 싸우지 않는다.
- oxfmt는 비교적 신생 도구다. 실제로 재현 불가능한 포맷/크래시 이슈가 나오면 임의로 Prettier로 갈아타지 말고 [tech-stack.md](tech-stack.md) 한계로 보고하고 사용자와 논의한다.

## 아키텍처 규칙

- **편집 엔진은 `packages/editor` 안에 가둔다.** CM6(`@codemirror/*`) API를 앱 레이어(`apps/desktop/src/features`, `widgets`)에 직접 노출하지 않는다. 앱은 Editor Facade의 좁은 인터페이스만 쓴다.
- **진실의 원천은 마크다운 텍스트다**([decisions.md ADR-003](decisions.md)). 프리뷰/리딩은 파생 뷰다. 텍스트를 우회해 트리를 원천으로 삼는 상태를 만들지 않는다.
- **메인 스레드에서 전체 마크다운 렌더를 돌리지 않는다.** 리딩 뷰/익스포트 렌더는 Worker로 보낸다([parser-pipeline.md](parser-pipeline.md)).
- FSD 레이어 import 방향을 지킨다(상위→하위만). 위반은 `steiger`로 걸린다([frontend-fsd.md](frontend-fsd.md)).
- `src-tauri`는 얇은 IPC 어댑터로 두고 실질 로직은 `crates/*`에 둔다.

## 보안

- HTML 렌더 경로는 `rehype-sanitize`를 기본 on으로 두고 최종 HTML 주입 전 안전 경계로 사용한다. 신뢰할 수 없는 마크다운의 raw HTML/스크립트를 그대로 실행하지 않는다([security-model.md](security-model.md)).
- Tauri IPC 커맨드는 경로 검증을 거친다. 워크스페이스 밖 임의 경로 read/write를 무분별하게 허용하지 않는다(`crates/fs-engine`의 경로 검증 단일 출처).
- 파일 저장은 원자적으로 한다(temp 파일 write → rename). 저장 중 크래시가 사용자 문서를 손상시키지 않게 한다.

## 문서와 설명

- 새 코드에는 의도를 설명하는 주석을 남긴다. 주석은 코드가 *무엇을* 하는지보다 *왜* 존재하는지를 설명한다.
- 구현을 진행할 때마다 관련 `docs/` 문서가 실제 코드와 맞는지 확인한다. 달라지면 [PR 체크리스트](pr-checklist.md)에 따라 문서를 먼저 갱신한다.
- 설계/전략 변경은 코드보다 [decisions.md](decisions.md)를 먼저 고친다.

## 전략 유지

- 모든 PR은 기존 전략(성능 예산·아키텍처 경계·원천 모델)에 지장이 없는지 평가한다. 전략을 수정해야 하면 임의로 바꾸지 말고 사용자와 먼저 논의한다.
- 한계를 숨기고 구현을 밀어붙이지 않는다. 예상 못 한 한계나 문서와의 차이가 드러나면 즉시 사용자에게 보고한다.
- **Linux WebKitGTK 관련 렌더/성능 문제**는 프로토타입 이후 플랫폼 검증 단계에서 발견 즉시 보고한다(셸 선택의 알려진 리스크, [tauri-shell.md](tauri-shell.md)).

## 버그 수정

- 버그는 증상이 아니라 루트커즈를 고친다.
- 구조가 원인이면 구조를 바꾼다.
- 임시 패치가 필요할 때도 왜 임시인지, 루트커즈 수정 계획이 무엇인지 보고한다.

## 테스트와 E2E

- 동작을 구현 전에 표현할 수 있는 영역은 TDD를 기본값으로 둔다.
- 모든 테스트 파일은 그 테스트가 무엇을 증명하는지, 그리고 편집 경험에서 왜 중요한지 설명해야 한다.
- 모든 기능 영역에는 E2E 또는 명시적 수동 검증 경로가 있어야 한다.
- 자동 E2E가 불가능한 영역(예: **macOS는 tauri-driver 미지원**)은 완료 처리 전에 이유와 수동 검증 방법을 사용자에게 보고한다([testing-strategy.md](testing-strategy.md)).
- 편집 hot path(타이핑·라이브 프리뷰·대용량 문서)는 성능 회귀를 [성능 예산](performance-budget.md)의 벤치로 함께 확인한다.
- **PR 올리기 전 pre-push 훅(린트 + 유닛 테스트)이 반드시 통과해야 한다**([development-commands.md Git 훅](development-commands.md)). 강제는 로컬 훅이 하고, CI는 최소 유닛 테스트만 둔다(나머지 백로그).
- 개발·검증 1차 타깃은 **macOS 로컬(수동 실행)** 이다. Windows/Linux 자동화는 백로그.

## 성능

- 성능은 이 제품의 핵심 약속이다. "Obsidian/Typora보다 빠르다"를 감이 아니라 **p50/p95/p99 수치**로 증명한다([performance-budget.md](performance-budget.md)).
- hot path에 눈에 띄는 회귀를 넣는 PR은 근거와 완화책을 함께 보고한다.
