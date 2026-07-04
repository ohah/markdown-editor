# Tauri 셸 경계 (Rust 백엔드)

Tauri 2.x가 데스크탑 셸이다. Rust 백엔드는 파일 I/O·워칭·검색·창/메뉴를 담당하고, **UI 상태를 소유하지 않는다**(복구용 메타데이터만). `src-tauri`는 얇은 IPC 어댑터로 두고 실질 로직은 `crates/*`에 둔다([project-structure.md](project-structure.md)).

## 책임 분리

```text
웹뷰(프론트, TS):
  UI 상태, 편집(CM6), 프리뷰 렌더(Worker)
  파일 내용의 소유자(진실의 원천 = 열린 문서 텍스트)

Rust 백엔드:
  디스크 접근(read/write/watch), 워크스페이스 검색, 창/메뉴/트레이
  UI 상태를 저장하지 않음. 결과를 IPC로 전달만.
```

## IPC 계약

- 프론트-Rust 계약의 **TS 측 타입 단일 출처는 `packages/ipc`** 다. 커맨드 이름·인자·반환 타입을 여기 모은다.
- 커맨드는 명령형 요청(`open_file`, `save_file`, `list_workspace`, `search_workspace`), 이벤트는 백엔드→프론트 알림(`file_changed`, `watch_event`)으로 나눈다.

```text
commands(invoke):
  open_file(path) -> { text, encoding, mtime }
  save_file(path, text) -> { mtime }             // 원자적 저장
  list_workspace(root) -> FileTree
  search_workspace(root, query) -> Matches

events(emit):
  file_changed(path, kind)                        // 외부 편집 감지
  watch_error(message)
```

## 파일 I/O 규칙

- **원자적 저장**: temp 파일에 write → fsync → rename. 저장 중 크래시가 원본을 손상시키지 않게 한다(`crates/fs-engine` 단일 출처).
- **인코딩**: UTF-8 기본. BOM/인코딩 감지 결과를 열 때 메타로 돌려주고, 저장 시 원래 인코딩을 보존한다.
- **경로 검증**: 워크스페이스 밖 임의 경로 접근을 무분별하게 허용하지 않는다. 경로 정규화·탈출 방지는 `fs-engine`에서 단일 출처로 검증한다([project-rules.md 보안](project-rules.md)).
- **외부 편집 감지**: 워처(`notify`)가 파일 변경을 감지해 `file_changed` 이벤트를 보낸다. 프론트는 dirty 상태와 충돌하면 사용자에게 알린다(reload/keep).

## 검색

- 워크스페이스 전문 검색은 Rust에서 ripgrep식으로 처리한다(`crates/workspace-index`). 대용량 vault에서 JS 순회보다 빠르다.
- 검색은 순수 로직으로 두어 `cargo test`로 창 없이 검증한다.

## 플랫폼 웹뷰 — 알려진 리스크(중요)

Tauri는 OS 웹뷰를 쓰므로 플랫폼마다 렌더 엔진이 다르다.

| OS | 웹뷰 | 상태 |
| --- | --- | --- |
| macOS | WKWebView | 안정. 단 WebDriver 미지원 → 자동 E2E 불가([testing-strategy.md](testing-strategy.md)). |
| Windows | WebView2(Chromium) | 안정. Chromium 기반이라 렌더 예측 가능. |
| Linux | **WebKitGTK** | **약한 고리.** 렌더 버그·스크롤·WebGPU/WebGL 미성숙 가능. |

**정책**: 앱 실행 프로토타입까지는 macOS 로컬 실행만 완료 기준으로 둔다([product-scope.md](product-scope.md)). 프로토타입 이후 Windows/Linux 검증 단계에서 Linux WebKitGTK를 우선 확인한다. 특히 CM6 라이브 프리뷰 decoration, 스크롤, 폰트 렌더가 리눅스에서 깨지거나 느리지 않은지 확인하고, 문제가 나오면 즉시 한계로 보고한다([project-rules.md 전략 유지](project-rules.md)).

Linux 빌드에는 시스템 의존성(webkit2gtk, gtk, librsvg, patchelf 등)이 필요하다. 정확한 목록은 스캐폴딩 시 이 문서에 배포판별로 기록한다.

## 창·메뉴·업데이트

- 네이티브 메뉴/단축키는 Rust 쪽에서 정의하되, 액션은 IPC로 프론트 커맨드에 매핑한다(단축키 로직 단일화).
- 자동 업데이트·서명·배포 전략은 후속 문서(`distribution.md`)로 분리한다(초기 범위 밖).

## 관측 가능성

- IPC 요청/이벤트를 구조화 로그로 남겨, "파일이 안 열림/저장 실패"가 프론트·IPC·Rust 중 어디서 깨졌는지 분리한다([architecture.md 관측 가능성](architecture.md)).
- 민감정보(홈 경로·파일 내용)는 로그에서 일반화/제거한다.
