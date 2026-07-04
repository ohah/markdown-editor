# 파일/폴더 구조

이 에디터는 **모노레포**다. JS/TS는 Bun 워크스페이스, Rust는 Cargo 워크스페이스로 한 리포에서 관리한다. 편집 엔진·마크다운 파이프라인·UI를 앱과 분리된 패키지로 두어 경계를 명확히 하고 교체 가능성을 지킨다([decisions.md ADR-002/005](decisions.md)).

## 원칙

- 앱(`apps/desktop`)은 얇게 두고, 재사용 로직은 `packages/*`·`crates/*`로 내린다.
- 편집 엔진은 `packages/editor` 하나에 가둔다. CM6 API를 앱 전역에 흩뿌리지 않는다.
- 파일은 가능한 한 하나의 목적을 갖는다. 서로 다른 이유로 바뀌는 코드가 한 파일에 쌓이면 공개 API를 유지한 채 분리한다.
- 프론트엔드 내부는 FSD 레이어 규칙을 따른다([frontend-fsd.md](frontend-fsd.md)).
- 빈 폴더도 `README`나 주석으로 의도를 남겨 나중의 위치 재조정을 줄인다.

## 최상위 구조

```text
markdown-editor/
  AGENTS.md                에이전트 진입점(인덱스)
  CLAUDE.md                Claude 진입점(포인터)
  docs/                    설계 문서(단일 출처)
  .mise.toml               개발 도구 버전 + 태스크(mise run) 단일 출처
  .githooks/               커밋된 git 훅(pre-commit/pre-push), core.hooksPath 대상
  package.json             Bun 워크스페이스 루트 + 스크립트
  bunfig.toml              Bun 설정
  tsconfig.base.json       공용 TS 설정
  .oxlintrc.json           oxlint 규칙
  oxfmt 설정               (oxfmt 설정 파일 형식은 스캐폴딩 시 확정)
  Cargo.toml               Rust 워크스페이스 루트
  .github/workflows/       CI(최소 유닛 테스트 — 나머지 백로그)
  apps/
    desktop/               Tauri 앱(웹 프론트 + Rust 셸)
  packages/                재사용 TS 패키지
  crates/                  재사용 Rust crate
```

## 앱 구조 (`apps/desktop`)

```text
apps/desktop/
  index.html
  vite.config.ts
  package.json
  src/                     React 프론트엔드(FSD)
    app/                   앱 초기화: providers, router, 전역 스타일, 테마
    pages/                 라우트/화면 단위(workbench 등)
    widgets/               복합 UI 블록: sidebar, editor-pane, statusbar, command-palette, preview-pane
    features/              사용자 상호작용: open-file, save-file, toggle-preview, search-workspace, switch-view-mode
    entities/              도메인 엔티티: document, workspace(vault), note
    shared/                ui 킷, lib, config, api(Tauri IPC 클라이언트)
  src-tauri/               Rust 백엔드(Tauri crate)
    Cargo.toml
    tauri.conf.json
    src/
      main.rs              엔트리
      commands/            IPC 커맨드(파일 open/save/list/search)
      fs/                  파일 I/O·원자적 저장(crates/fs-engine 소비)
      watch/               파일 워처(notify)
      menu/                네이티브 메뉴/단축키
```

FSD 레이어 안의 슬라이스는 `ui/model/api/lib/config` 세그먼트로 나눈다. import 방향과 금지 규칙은 [frontend-fsd.md](frontend-fsd.md)를 단일 출처로 둔다.

## 패키지 구조 (`packages/*`)

```text
packages/
  core/                    Document Model(텍스트=진실의 원천), 소스맵, dirty 상태 — 프레임워크 중립
  editor/                  CodeMirror 6 래핑(Editor Facade): EditorState/View, 키맵, 마크다운 확장,
                           라이브 프리뷰 decoration. 앱엔 좁은 인터페이스만 노출(교체 가능)
  markdown/                unified 파이프라인(remark→rehype→sanitize) 순수 코어 + Worker 진입점
  ui/                      공용 디자인 시스템/컴포넌트(React) — 앱과 무관하게 테스트 가능
  ipc/                     Tauri IPC 타입/클라이언트(프론트-Rust 계약의 TS 측 단일 출처)
```

`packages/editor`는 이 프로젝트의 성능·교체 전략의 핵심이다. 나중에 편집 표면을 B(캔버스)로 바꾸더라도 이 패키지의 인터페이스만 다시 구현한다.

## Crate 구조 (`crates/*`)

```text
crates/
  fs-engine/               원자적 저장(temp+rename), 인코딩 감지, 경로 검증 — src-tauri가 소비
  workspace-index/         워크스페이스(vault) 파일 인덱싱·검색(ripgrep식) — 순수 로직, 테스트 가능
```

`src-tauri`는 얇은 IPC 어댑터로 두고, 실질 로직을 위 crate에 두어 `cargo test`로 창 없이 검증할 수 있게 한다.

## 문서 구조

```text
docs/
  tech-stack.md            확정 스택·버전 정책
  decisions.md             ADR(왜 이 선택인가)
  architecture.md          레이어·경계·스레딩
  project-structure.md     이 문서
  project-rules.md         필수 규칙
  development-commands.md  개발 명령
  editor-strategy.md       CM6·라이브 프리뷰
  parser-pipeline.md       unified 파이프라인
  frontend-fsd.md          FSD 레이어 규칙
  tauri-shell.md           Rust 백엔드 경계
  performance-budget.md    성능 예산·벤치
  testing-strategy.md      테스트·E2E·커버리지
  verification-matrix.md   무엇을 무엇으로 검증하는가
  implementation-plan.md   구현 순서
  initial-vertical-slice.md 첫 세로 슬라이스
  pr-checklist.md          PR 규칙
  references.md            레퍼런스·clean-room
```

## 테스트 배치 규칙

- TS 단위 테스트는 각 패키지/슬라이스 옆에 둔다(`*.test.ts(x)`), 커버리지는 Vitest v8.
- Rust 단위 테스트는 crate 내부(`#[cfg(test)]`)에 둔다.
- E2E는 `apps/desktop/e2e/`(WebdriverIO + tauri-driver)에 둔다.
- 새 테스트 경로를 추가하면 같은 PR에서 루트 스크립트·CI 워크플로에 연결한다. 자동화가 불가능하면 그 이유와 수동 검증 방법을 PR에 남긴다([verification-matrix.md](verification-matrix.md)).
