# 테스트 전략 (단위 · E2E · 커버리지)

가능한 모든 영역에서 TDD를 기본값으로 둔다. 목표는 "테스트 개수"가 아니라, 구현 전에 원하는 편집 동작을 작게 고정해 코드가 커져도 책임 경계가 흐려지지 않게 하는 것이다. 단위 커버리지는 넓게 확보하되, **검증 우선순위와 강제 지점은 다음과 같이 둔다.**

- **개발·검증은 macOS 우선(로컬)**: 1차 개발과 실제 동작 확인은 맥 환경에서 로컬로 한다. macOS는 tauri-driver E2E가 안 되므로(아래 "E2E의 플랫폼 한계") 앱 동작은 **로컬 수동 실행**으로 확인한다. Windows/Linux 검증은 후속(백로그).
- **강제 게이트는 로컬 git 훅**: PR 올리기 전 린트·유닛 테스트를 **pre-push 훅에서 반드시 실행**한다(아래 "로컬 강제 게이트"). CI에 의존하지 않고 로컬에서 막는다.
- **CI는 최소**: 지금은 간단한 유닛 테스트만 돌린다. 크로스 플랫폼 E2E·성능·오라클 CI는 **백로그**다(아래 "CI 범위").

## 레이어별 테스트

```text
순수 단위(가장 많이):
  packages/core      문서 모델(텍스트=원천, dirty, 소스맵)
  packages/markdown  unified core.ts: 입력 md -> 기대 HTML/headings/stats, sanitize 회귀
  packages/editor    라이브 프리뷰 decoration 계산(순수 함수): 문서+선택 -> decoration 집합
  crates/fs-engine   원자적 저장, 경로 검증, 인코딩
  crates/workspace-index  검색 로직

컴포넌트/통합:
  packages/ui, apps/desktop/src/**  React Testing Library + Vitest(jsdom/happy-dom)
  Worker 계약(postMessage, stale 취소) 클라이언트 테스트
  IPC 클라이언트(packages/ipc) 모킹 테스트

E2E(가장 적게, 가장 비쌈):
  apps/desktop/e2e   WebdriverIO + tauri-driver
    open -> edit -> save 왕복
    소스/라이브/리딩 뷰 전환
    검색 -> 결과 열기
```

## 도구

| 대상 | 러너 | 커버리지 |
| --- | --- | --- |
| TS(프론트 + packages) | **Vitest** | v8 (`bun run test:cov`) |
| Rust(crates + src-tauri 로직) | **cargo test** | `cargo-llvm-cov`(옵션) |
| E2E | **WebdriverIO + tauri-driver** | — |

## E2E의 플랫폼 한계(중요)

`tauri-driver`는 WebDriver로 Tauri 앱을 구동하는데, **macOS WKWebView는 WebDriver를 지원하지 않는다**. 따라서:

```text
Linux   : E2E 자동화 O (WebKitWebDriver)
Windows : E2E 자동화 O (Edge WebDriver / WebView2)
macOS   : E2E 자동화 X → 컴포넌트/통합 테스트 + 수동 검증으로 대체
```

이는 숨기지 않고 [verification-matrix.md](verification-matrix.md)와 PR에 한계로 명시한다. macOS 전용 회귀(WKWebView 렌더 차이 등)는 수동 검증 시나리오로 남긴다.

## 자동화하기 어려운 영역

다음은 자동 E2E로 완전히 잡기 어렵다. 완료 처리 전에 이유와 수동 검증 방법을 사용자에게 보고한다([project-rules.md](project-rules.md)).

- **한글 IME 조합**: composition 이벤트를 WebDriver로 충실히 재현하기 어렵다. → 수동 시나리오(조합 중 프리뷰 미변경, 마지막 자모 삭제, 확정 후 커서 이동)를 [editor-strategy.md IME](editor-strategy.md)와 함께 문서화하고, 가능하면 CM6 트랜잭션 레벨 단위 테스트로 근사한다.
- **플랫폼 웹뷰 렌더 차이**(특히 Linux WebKitGTK): 스크린샷 수동 확인 + 초기 세로 슬라이스에서 조기 점검.
- **성능**: 회귀 방향은 벤치로 보되 절대 pass/fail 게이트는 아니다([performance-budget.md](performance-budget.md)).

## 커버리지 정책

- 순수 로직(core/markdown/editor decoration/fs-engine/workspace-index)은 커버리지를 높게 유지한다 — 여기가 회귀가 가장 아픈 곳이다.
- UI/E2E는 핵심 사용자 흐름(열기·편집·저장·뷰 전환·검색)을 우선 커버한다. 100%를 위한 커버리지를 쫓지 않는다.
- 커버리지 리포트는 CI 아티팩트로 남긴다.

## TDD 적용 원칙

- 동작을 구현 전에 표현할 수 있으면 실패하는 테스트를 먼저 쓴다(특히 파서 출력, decoration 계산, 원자적 저장, 경로 검증).
- 각 테스트는 "이 동작이 사용자의 어떤 편집 경험을 지키는가"를 파일 주석으로 남긴다.
- 버그 수정은 재현 테스트를 먼저 추가하고 루트커즈를 고친다([project-rules.md 버그 수정](project-rules.md)).

## 로컬 강제 게이트 (git hooks)

강제 지점은 CI가 아니라 **로컬 git 훅**이다. `core.hooksPath = .githooks`로 리포에 커밋된 훅을 쓴다(별도 도구 의존성 없이 셸 스크립트).

```text
.githooks/pre-commit   빠른 피드백: oxlint + oxfmt 검사 + cargo fmt 검사
.githooks/pre-push     PR 전 필수 게이트: oxlint + vitest(유닛) + clippy + cargo test
```

- **pre-push가 "PR 올리기 전 반드시 실행"의 실체다.** 린트와 유닛 테스트가 통과하지 않으면 push가 막힌다.
- 훅은 툴체인이 아직 없으면(docs-only 단계) 조용히 건너뛰고, 스캐폴딩 후 자동으로 활성화된다.
- 새 클론에서 훅을 켜려면 `git config core.hooksPath .githooks`(스캐폴딩 후엔 `bun install`의 `prepare`가 자동 설정). 명령은 [development-commands.md Git 훅](development-commands.md).
- 긴급 시 `git push --no-verify`로 우회할 수 있으나 기본적으로 쓰지 않는다.

## CI 범위 (최소 · 나머지는 백로그)

강제는 로컬 훅이 하므로 CI는 얇게 둔다. 지금은 **간단한 유닛 테스트만** 돌린다.

```text
.github/workflows/ci.yml (현재 범위):
  - vitest run (유닛 테스트)
  - (선택) oxlint

백로그(나중에 추가):
  - cargo test / cargo clippy
  - 크로스 플랫폼 E2E (tauri-driver, Linux/Windows)
  - 성능 벤치 리포트
  - 커버리지 아티팩트
```

새 테스트 경로는 우선 로컬 훅에서 돌게 하고, CI 확장은 백로그 우선순위에 따라 추가한다.
