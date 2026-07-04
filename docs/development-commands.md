# 개발 명령

이 에디터는 **mise**로 개발 도구 버전을 고정하고 태스크를 실행한다. `mise run <task>`가 기본 인터페이스이고, 내부적으로 bun 스크립트와 cargo를 호출한다. 도구·버전·태스크 정의의 단일 출처는 리포 루트의 `.mise.toml`이다(필요하면 `bun run …`/`cargo …`를 직접 호출해도 된다).

> 아래는 계획된 태스크 규약이다. 스캐폴딩 PR에서 실제 스크립트로 연결하고, 달라진 부분은 `.mise.toml`과 이 문서를 함께 갱신한다.

## 최초 세팅

```sh
mise install                          # .mise.toml의 도구(bun, rust + rustfmt/clippy) 설치
git config core.hooksPath .githooks   # git 훅 활성화(스캐폴딩 후엔 자동)
```

mise가 없으면 먼저 설치한다(https://mise.jdx.dev). 도구 버전은 `.mise.toml`이 단일 출처다.

## 개발 실행

```sh
mise run dev           # Tauri dev(웹 dev 서버 + Rust 백엔드 핫리로드)
mise run dev-web       # 프론트만 Vite dev(웹뷰 없이 UI 확인)
```

## 빌드

```sh
mise run build         # 프로덕션 번들 + Tauri 앱 빌드(현재 OS 타깃)
mise run build-web     # 프론트만 빌드(정적 산출물 확인)
```

## 린트 / 포맷

```sh
mise run lint          # oxlint (JS/TS)
mise run fmt           # oxfmt 적용(JS/TS) + cargo fmt(Rust)
mise run fmt-check     # 포맷 검사(변경 없이): oxfmt + cargo fmt --check
mise run fsd-check     # steiger — FSD 레이어 import 경계 검사
mise run clippy        # cargo clippy(경고=에러)
```

## 테스트

```sh
mise run test          # Vitest 유닛(= vitest run) + cargo test
mise run test-watch    # Vitest 워치 모드(개발 중)
mise run test-cov      # 커버리지: Vitest v8 (+ cargo llvm-cov 옵션)
mise run e2e           # WebdriverIO + tauri-driver (Linux/Windows)
```

**macOS E2E 주의**: macOS는 자동 E2E가 안 되어 로컬 수동 검증으로 대체한다. 이유(WKWebView WebDriver 미지원)와 대체 경로의 단일 출처는 [testing-strategy.md E2E의 플랫폼 한계](testing-strategy.md)다.

## 성능

```sh
mise run bench         # 입력 레이턴시/시작/메모리 벤치(p50/p95/p99)
```

벤치 방법론과 워터마크(실기기 없이 방향성만)는 [성능 예산](performance-budget.md)을 단일 출처로 둔다.

## 전체 확인 게이트

```sh
mise run check         # fmt-check + lint + fsd-check + test + clippy 를 한 번에
git diff --check       # 공백/충돌 마커 확인
```

`mise run check`는 `.mise.toml`의 `depends`로 하위 태스크를 묶는다. 개별 태스크(`mise run lint` 등)를 따로 돌려도 된다.

## Git 훅 (PR 전 필수 게이트)

강제 검증은 CI가 아니라 로컬 git 훅에서 한다([testing-strategy.md 로컬 강제 게이트](testing-strategy.md)). 리포에 커밋된 `.githooks/`를 `core.hooksPath`로 쓰며, 훅은 `mise run` 태스크를 호출한다.

```sh
# 훅 활성화(새 클론에서 1회 — 스캐폴딩 후엔 mise/prepare가 자동 설정)
git config core.hooksPath .githooks

# 훅이 도는 검사
#   pre-commit : mise run lint + mise run fmt-check   (빠른 피드백)
#   pre-push   : mise run check                       (PR 전 필수 게이트)
```

- pre-push가 통과하지 않으면 push가 막힌다 — "PR 올리기 전 반드시 실행".
- 툴체인 미설정(docs-only) 단계에선 훅이 조용히 건너뛴다. 스캐폴딩 후 자동 활성화.
- 긴급 우회: `git push --no-verify`(기본적으로 쓰지 않는다).

## 완료 전 확인

코드/빌드 설정을 바꾼 PR은 위 **전체 확인 게이트**(`mise run check` + `git diff --check`)를 통과해야 한다. 이 중 린트·유닛 테스트는 pre-push 훅이 자동으로 강제한다(위 Git 훅).

문서만 바꾼 PR은 `git diff --check`를 최소 검증으로 쓴다. 다만 문서가 명령·구조·테스트 경로를 바꾸면 관련 명령도 함께 실행한다.

## 플랫폼별 검증

앱 실행 프로토타입까지는 macOS 로컬 실행만 완료 기준으로 둔다([product-scope.md](product-scope.md)). Tauri 웹뷰 차이 때문에 프로토타입 이후 Windows/Linux를 별도 단계에서 확인하며, 그때 **Linux WebKitGTK**를 우선 확인한다(상세와 시스템 의존성 목록은 [tauri-shell.md](tauri-shell.md)).

```sh
mise run dev           # 각 OS에서 CM6 편집/라이브 프리뷰 렌더 확인
```
