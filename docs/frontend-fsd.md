# 프론트엔드 구조 (Feature-Sliced Design)

`apps/desktop/src`는 Feature-Sliced Design(FSD)을 따른다. 목적은 기능이 빠르게 늘어나는 에디터에서 **단방향 의존**을 강제해 스파게티를 막는 것이다([decisions.md ADR-006](decisions.md)).

## 레이어 (위 → 아래로만 의존)

```text
app        앱 초기화: providers, router, 전역 스타일/테마, 에러 바운더리
pages      화면/라우트 단위 조립(workbench 등)
widgets    복합 UI 블록: sidebar, editor-pane, preview-pane, statusbar, command-palette
features   사용자 상호작용(동사): open-file, save-file, toggle-preview, switch-view-mode, search-workspace
entities   도메인 엔티티(명사): document, workspace(vault), note
shared     ui 킷, lib(순수 유틸), config, api(Tauri IPC 클라이언트)
```

**의존 규칙**: 위 레이어는 아래 레이어만 import 한다. 같은 레이어의 다른 슬라이스를 직접 import 하지 않는다(교차 의존 금지). 예: `features/save-file`은 `entities/document`, `shared/*`를 쓰지만 `features/open-file`을 직접 쓰지 않는다.

## 슬라이스와 세그먼트

각 레이어는 도메인별 **슬라이스**로 나뉘고, 슬라이스는 **세그먼트**로 나뉜다.

```text
features/
  save-file/
    ui/       React 컴포넌트(버튼/단축키 핸들러)
    model/    상태/로직(store, 이벤트, 커맨드)
    api/      바깥 통신(Tauri IPC 호출은 shared/api 경유)
    lib/      슬라이스 로컬 순수 유틸
    index.ts  공개 API(배럴) — 바깥은 이 파일로만 접근
```

- 슬라이스의 **공개 API는 `index.ts`(배럴)** 뿐이다. 슬라이스 내부 파일을 바깥에서 깊은 경로로 import 하지 않는다.
- 세그먼트는 필요한 것만 둔다(모든 슬라이스가 4개를 다 갖지 않는다).

## 이 에디터에서의 배치 예시

```text
entities/
  document/     문서 엔티티: 현재 열린 문서(텍스트=원천), dirty, 경로, 통계
  workspace/    vault(폴더) 엔티티: 파일 트리, 현재 워크스페이스
  note/         (선택) 노트 메타/링크

features/
  open-file/        파일 열기(IPC) → document 갱신
  save-file/        저장(원자적, IPC) ← document
  switch-view-mode/ 소스/라이브/리딩 전환(EditorFacade 호출)
  toggle-preview/   리딩 뷰 패널 토글(Worker 렌더)
  search-workspace/ 워크스페이스 검색(IPC, ripgrep식)

widgets/
  editor-pane/      packages/editor(EditorFacade)를 감싸는 위젯
  preview-pane/     packages/markdown Worker 결과를 렌더
  sidebar/          파일 트리 + 검색
  statusbar/        커서/문자수/뷰 모드
  command-palette/  커맨드 실행

shared/
  api/              Tauri IPC 클라이언트(packages/ipc 타입 사용)
  ui/               버튼/패널 등(무거운 것은 packages/ui로)
  lib/              순수 유틸(경로, 포맷)
  config/           앱 설정 스키마/로딩
```

## packages와의 관계

- `packages/editor`(EditorFacade), `packages/markdown`(Worker), `packages/ui`, `packages/ipc`, `packages/core`는 앱 바깥의 재사용 코드다.
- `widgets/editor-pane`은 `packages/editor`를, `widgets/preview-pane`은 `packages/markdown`을 소비한다.
- 즉 **무거운 도메인 로직은 packages에**, **화면 조립·상호작용은 FSD 레이어에** 둔다.

## 경계 강제

- `steiger`로 FSD 레이어/슬라이스 import 방향을 자동 검사한다(`mise run fsd-check`). 이는 maru의 facade 경계 검사와 같은 역할이다([verification-matrix.md](verification-matrix.md)).
- 위반이 나오면 플래그로 우회하지 말고 슬라이스를 올바른 레이어로 옮기거나 공개 API를 정리한다.

## 규칙 요약

- import는 항상 아래 레이어로, 슬라이스는 `index.ts`로만.
- CM6·unified 세부는 packages 뒤에 숨긴다(앱은 facade만).
- 한 슬라이스가 서로 무관한 책임을 갖기 시작하면 플래그 대신 슬라이스를 분리한다.
