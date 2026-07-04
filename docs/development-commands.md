# 개발 명령

이 에디터 작업에서 사용하는 기본 명령이다. 실제 스크립트 이름은 스캐폴딩 시 루트 `package.json`과 `Cargo.toml`에 확정하고, 이 문서를 단일 출처로 유지한다.

> 아래는 계획된 명령 규약이다. 스캐폴딩 PR에서 실제 스크립트로 연결하고, 달라진 부분은 이 문서를 갱신한다.

## 부트스트랩

```sh
bun install            # JS/TS 워크스페이스 의존성 설치
cargo fetch            # Rust 의존성 사전 페치(선택)
```

## 개발 실행

```sh
bun run dev            # Tauri dev(웹 dev 서버 + Rust 백엔드 핫리로드)
bun run dev:web        # 프론트만 Vite dev(웹뷰 없이 UI 확인)
```

## 빌드

```sh
bun run build          # 프로덕션 번들 + Tauri 앱 빌드(현재 OS 타깃)
bun run build:web      # 프론트만 빌드(정적 산출물 확인)
```

## 린트 / 포맷

```sh
bun run lint           # oxlint (JS/TS)
bun run fmt            # oxfmt 적용 (JS/TS)
bun run fmt:check      # oxfmt 검사(변경 없이)
bun run fsd:check      # steiger — FSD 레이어 import 경계 검사(옵션)

cargo clippy --all-targets --all-features -- -D warnings   # Rust 린트(경고=에러)
cargo fmt --check                                          # Rust 포맷 검사
```

## 테스트

```sh
bun run test           # Vitest 단위 테스트(= vitest run, 워치 아님 — 훅/CI용)
bun run test:watch     # Vitest 워치 모드(개발 중)
bun run test:cov       # Vitest 커버리지(v8)
cargo test             # Rust 단위/통합 테스트
cargo llvm-cov         # Rust 커버리지(옵션)

bun run e2e            # WebdriverIO + tauri-driver (Linux/Windows)
```

**macOS E2E 주의**: `tauri-driver`는 WKWebView WebDriver 미지원이라 macOS에서 자동 E2E가 안 된다. macOS는 컴포넌트/통합 테스트 + 수동 검증으로 대체한다([testing-strategy.md](testing-strategy.md)).

## 성능

```sh
bun run bench          # 입력 레이턴시/시작/메모리 벤치 하니스(p50/p95/p99)
```

벤치 방법론과 워터마크(실기기 없이 방향성만)는 [성능 예산](performance-budget.md)을 단일 출처로 둔다.

## 전체 확인 게이트

```sh
bun run check          # fmt:check + lint + fsd:check + test + (web)build 를 한 번에
cargo clippy --all-targets -- -D warnings && cargo fmt --check && cargo test
git diff --check       # 공백/충돌 마커 확인
```

## Git 훅 (PR 전 필수 게이트)

강제 검증은 CI가 아니라 로컬 git 훅에서 한다([testing-strategy.md 로컬 강제 게이트](testing-strategy.md)). 리포에 커밋된 `.githooks/`를 `core.hooksPath`로 쓴다.

```sh
# 훅 활성화(새 클론에서 1회 — 스캐폴딩 후엔 bun install의 prepare가 자동 실행)
git config core.hooksPath .githooks

# 훅이 도는 검사
#   pre-commit : oxlint + oxfmt 검사 + cargo fmt 검사        (빠른 피드백)
#   pre-push   : oxlint + vitest(유닛) + clippy + cargo test (PR 전 필수 게이트)
```

- pre-push가 통과하지 않으면 push가 막힌다 — "PR 올리기 전 반드시 실행".
- 툴체인 미설정(docs-only) 단계에선 훅이 조용히 건너뛴다. 스캐폴딩 후 자동 활성화.
- 긴급 우회: `git push --no-verify`(기본적으로 쓰지 않는다).

## 완료 전 확인

코드/빌드 설정을 바꾼 PR은 기본적으로 다음을 통과해야 한다.

```sh
bun run check
cargo clippy --all-targets -- -D warnings
cargo test
git diff --check
```

문서만 바꾼 PR은 `git diff --check`를 최소 검증으로 쓴다. 다만 문서가 명령·구조·테스트 경로를 바꾸면 관련 명령도 함께 실행한다.

## 플랫폼별 조기 검증

Tauri 웹뷰 차이 때문에 초기부터 3-OS를 굴린다. 특히 **Linux WebKitGTK**를 가장 먼저 확인한다.

```sh
# Linux 필수 시스템 의존성(예시, 배포판에 따라 다름)
#   webkit2gtk, libgtk, librsvg, patchelf 등 — 스캐폴딩 시 정확한 목록을 tauri-shell.md에 기록
bun run dev            # 리눅스에서 CM6 편집/라이브 프리뷰 렌더 확인
```
