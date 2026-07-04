# 실제 구현 계획

이 문서는 구현 순서의 단일 출처다. 원칙은 **경계를 먼저 고정하고, 얇은 세로 슬라이스로 실제 동작을 조기에 확인**하는 것이다. 특히 Tauri의 알려진 리스크(Linux WebKitGTK)를 뒤로 미루지 않는다.

## 단계 개요

```text
0. 스캐폴딩 / 툴체인
1. 최소 편집 세로 슬라이스 (열기·편집·저장)
2. 3-OS 조기 검증 (Linux WebKitGTK 최우선)
3. 뷰 모드 (소스 / 라이브 프리뷰 / 리딩)
4. 마크다운 파이프라인 (unified Worker)
5. 워크스페이스 (파일 트리 · 검색)
6. 성능 벤치 하니스
7. 다듬기 (설정 · 테마 · 단축키 · 배포)
```

각 단계는 [검증 매트릭스](verification-matrix.md)의 해당 행을 자동/수동 검증으로 채우며 끝낸다.

## 0. 스캐폴딩 / 툴체인

- Bun 워크스페이스 + Cargo 워크스페이스 모노레포([project-structure.md](project-structure.md)).
- Tauri 2 + Vite + React 19 앱(`apps/desktop`) 최소 창.
- oxlint / oxfmt / clippy / rustfmt / Vitest / cargo test 배선 + 루트 스크립트([development-commands.md](development-commands.md)).
- CI 스켈레톤(lint·fmt·test·build).
- `docs/tech-stack.md`의 "설치된 버전" 표를 실제 버전으로 채운다.
- **완료 기준**: `bun run check`와 `cargo clippy/test`가 빈 프로젝트에서 통과.

## 1. 최소 편집 세로 슬라이스

가장 얇게 "실제로 편집되는" 경로를 만든다. 상세는 [초기 세로 슬라이스](initial-vertical-slice.md).

- `packages/core`: 문서 모델(텍스트=원천, dirty).
- `packages/editor`: CM6 EditorFacade(마운트/문서 설정/변경 이벤트). 라이브 프리뷰는 아직 없음(소스 모드만).
- `src-tauri` + `crates/fs-engine`: `open_file`, `save_file`(원자적).
- `apps/desktop`: 열기→편집→저장 흐름(features/open-file, save-file; widgets/editor-pane).
- **완료 기준**: 파일을 열어 수정하고 저장하면 디스크에 반영. 단위 테스트(문서 모델·원자적 저장) + (가능 OS)E2E 왕복.

## 2. 3-OS 조기 검증 (리눅스 최우선)

- 세 OS에서 1단계 슬라이스를 실행한다. **Linux WebKitGTK를 가장 먼저** 확인한다([tauri-shell.md](tauri-shell.md)).
- CM6 편집·스크롤·폰트 렌더가 리눅스에서 정상인지, 성능이 급락하지 않는지 확인.
- 필요한 Linux 시스템 의존성을 `tauri-shell.md`에 배포판별로 기록.
- **완료 기준**: 3-OS에서 열기·편집·저장이 눈에 띄는 결함 없이 동작. 리눅스 이슈는 한계로 보고.

## 3. 뷰 모드

- `packages/editor/live-preview`: decoration 계산(순수) + ViewPlugin. 커서 줄 원문 노출, 조합(IME) 중 미변경([editor-strategy.md](editor-strategy.md)).
- `features/switch-view-mode`: 소스 ⟷ 라이브 ⟷ 리딩 전환.
- **완료 기준**: 세 뷰 전환이 원천 텍스트를 바꾸지 않고 동작. decoration 계산 단위 테스트. 한글 조합 수동 검증.

## 4. 마크다운 파이프라인 (Worker)

- `packages/markdown`: unified core(remark→rehype→sanitize) + Worker + 디바운스/stale 취소 클라이언트([parser-pipeline.md](parser-pipeline.md)).
- `widgets/preview-pane`: 리딩 뷰가 Worker 결과를 렌더(메인 스레드 블로킹 0).
- **완료 기준**: 리딩 뷰가 편집을 막지 않고 갱신. core 스냅샷 테스트 + sanitize 보안 테스트 + Worker 계약 테스트.

## 5. 워크스페이스

- `crates/workspace-index`: 파일 트리 + ripgrep식 검색.
- `entities/workspace`, `widgets/sidebar`, `features/search-workspace`.
- 외부 편집 감지(`notify` → `file_changed` 이벤트) 및 충돌 처리.
- **완료 기준**: 폴더 열기·트리·검색·결과 열기 동작. 검색/인덱스 `cargo test`. (가능 OS)E2E.

## 6. 성능 벤치 하니스

- `tools/bench`: 입력 레이턴시(p50/p95/p99), 콜드 스타트, 메모리. 워터마크·비교군([performance-budget.md](performance-budget.md)).
- 라이브 프리뷰 on/off, 한글 조합 시나리오 분리 측정.
- **완료 기준**: `bun run bench`가 리포트 생성. 경쟁작 비교군과 동일 시나리오 측정 가능.

## 7. 다듬기

- 설정(스키마 기반)·테마(라이트/다크)·단축키·커맨드 팔레트.
- 배포·서명·자동 업데이트는 별도 `distribution.md`로 분리(범위 밖).

## 진행 규칙

- 한 단계에서 다음 단계로 넘어갈 때 관련 문서가 코드와 맞는지 확인한다([pr-checklist.md](pr-checklist.md)).
- 설계와 실제가 달라지면 임의로 진행하지 말고 [decisions.md](decisions.md)를 갱신하고 사용자에게 보고한다.
