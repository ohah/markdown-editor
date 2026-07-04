# 검증 매트릭스

이 문서는 각 영역이 **무엇으로 검증되는지** 추적하는 표다. 목표는 "나중에 하자"가 아니라, 아직 구현되지 않은 영역도 어떤 테스트/산출물로 증명할지 먼저 정해 두는 것이다. 계획 단계이므로 대부분 "계획" 상태이며, 구현 시 상태와 산출물을 채운다.

## 자동 검증 대상 (계획)

| 영역 | 검증 방법 | 산출물 | 의미 |
| --- | --- | --- | --- |
| 문서 모델 | `bun run test` (Vitest, `packages/core`) | 커버리지 | 텍스트=원천, dirty 전이, 소스맵이 의도대로 동작한다. |
| 마크다운 렌더 | `bun run test` (`packages/markdown` core.ts) | 스냅샷 | 입력 md → 기대 HTML/headings/stats가 안정적이다. |
| sanitize 보안 | `bun run test` (보안 회귀) | — | 스크립트 주입 문서에서 스크립트가 제거된다(XSS 차단). |
| 라이브 프리뷰 decoration | `bun run test` (`packages/editor` 순수 계산) | — | 문서+선택 → decoration 집합이 정확하다(마크 숨김/렌더 적용, 커서 줄 원문 노출). |
| Worker 계약 | `bun run test` (client.ts) | — | postMessage 형식, 디바운스, stale 취소가 동작한다. |
| 원자적 저장 | `cargo test` (`crates/fs-engine`) | — | temp→rename 저장, 크래시 시 원본 보존, 인코딩 보존. |
| 경로 검증 | `cargo test` (`fs-engine`) | — | 워크스페이스 탈출 경로를 거부한다. |
| 워크스페이스 검색 | `cargo test` (`workspace-index`) | — | ripgrep식 검색 결과가 정확하다. |
| IPC 클라이언트 | `bun run test` (`packages/ipc` 모킹) | — | 커맨드 인자/반환 타입 계약이 지켜진다. |
| FSD 경계 | `bun run fsd:check` (steiger) | — | 레이어/슬라이스 import 방향 위반이 없다. |
| 린트/포맷 | `bun run lint`, `bun run fmt:check`, `cargo clippy -D warnings`, `cargo fmt --check` | — | 코드 스타일 게이트. |
| 컴포넌트/통합 | `bun run test` (RTL + Vitest) | — | 위젯·기능이 렌더/상호작용한다. |
| E2E(열기·편집·저장·전환·검색) | `bun run e2e` (tauri-driver, **Linux/Windows**) | 스크린샷/로그 | 실제 앱에서 핵심 흐름이 왕복한다(백로그, CI 강제 아님). macOS는 로컬 수동. |
| 프론트 빌드 | `bun run build:web` | 번들 | 프로덕션 번들이 성공한다. |
| Tauri 빌드 | `bun run build` | 앱 산출물 | 현재 OS 타깃으로 앱이 빌드된다. |

## 아직 완전 자동 검증이 아닌 영역

불가 이유는 다음 의미로 쓴다.

- `구현 전`: 기능/러너가 아직 없어 못 검증한다. 만들면 자동화할 수 있다.
- `환경 의존`: OS 웹뷰, GPU, 폰트 스택처럼 실행 환경에 따라 결과가 달라진다.
- `도구 한계`: 도구가 구조적으로 지원하지 않는다.

| 영역 | 현재 상태 | 이유 | 대체 검증 |
| --- | --- | --- | --- |
| macOS E2E | 불가 | 도구 한계(tauri-driver가 WKWebView WebDriver 미지원) | 컴포넌트/통합 테스트 + 수동 흐름 검증 |
| 한글 IME 조합 | 부분 | 도구 한계(WebDriver로 composition 재현 어려움) | CM6 트랜잭션 단위 테스트 근사 + 수동 시나리오 |
| Linux WebKitGTK 렌더 차이 | 부분 | 환경 의존 | 초기 세로 슬라이스 조기 점검 + 스크린샷 수동 확인 |
| 성능(절대값) | 방향성만 | 환경 의존(하드웨어 변동) | 벤치 리포트(p50/p95/p99, 워터마크), 회귀 방향 감시 |
| 리딩 뷰 시각 회귀 | 구현 전 | 러너 미구현 | 후속 스크린샷 골든 비교 도입 검토 |

## 관측 가능성 산출물

`*.html` 렌더 결과만으로는 내부 상태를 다 증명하지 못한다. 그래서 스냅샷은 도메인 데이터를 함께 남긴다.

- 문서 스냅샷: 텍스트, 선택, dirty, 뷰 모드
- 파서 스냅샷: mdast/hast 요약(노드 종류·개수), headings, stats
- decoration 스냅샷: 숨긴 마크 범위, 적용한 렌더 범위
- IPC 이벤트 로그: 요청/이벤트 순서(민감정보 제거)

이 포맷은 테스트 산출물이면서, 나중에 디버그 inspector가 같은 데이터를 보게 하는 첫 관측 가능성 경계다([architecture.md 관측 가능성](architecture.md)).

## 강제 지점 (로컬 훅 · CI 최소)

검증의 **강제 지점은 로컬 git 훅**이다(CI 아님). PR 전 pre-push가 린트·유닛 테스트를 반드시 돌린다([testing-strategy.md 로컬 강제 게이트](testing-strategy.md)).

- pre-commit: oxlint + oxfmt 검사 + cargo fmt 검사 (빠른 피드백)
- **pre-push(필수): oxlint + vitest(유닛) + clippy + cargo test** — 통과 못 하면 push 차단
- CI: 지금은 유닛 테스트만(나머지 백로그 — 크로스 플랫폼 E2E·성능·커버리지)
- 개발·검증 1차 타깃은 **macOS 로컬**(수동 실행). Windows/Linux 자동화는 백로그.

## 연결 규칙

새 테스트 파일을 추가하면 같은 PR에서 루트 스크립트·CI에 연결한다. 자동화가 불가능하면 문서와 PR에 다음을 남긴다: 왜 자동화가 불가능한가, 어떤 수동 검증을 했는가, 나중에 자동화하려면 어떤 경계가 필요한가.
