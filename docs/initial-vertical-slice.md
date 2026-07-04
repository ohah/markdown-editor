# 초기 세로 슬라이스

첫 구현 목표는 macOS에서 "가장 얇게 실제로 편집되는 경로"를 세우는 것이다. 화려한 기능이 아니라, **앞으로 지켜야 할 경계(문서 원천·에디터 facade·IPC·원자적 저장)를 컴파일·테스트 가능한 형태로 고정**하는 것이 목적이다.

## 슬라이스 범위

```text
포함:
  - 파일 열기 (IPC -> 텍스트)
  - CM6 소스 모드 편집 (라이브 프리뷰 없음)
  - dirty 표시
  - 파일 저장 (원자적)
  - 최소 성능 벤치 smoke(콜드 스타트, 유휴 메모리, 입력 레이턴시)

제외(다음 단계):
  - 라이브 프리뷰 / 리딩 뷰
  - unified Worker
  - Windows/Linux 검증
  - 워크스페이스 / 검색 / 파일 트리
  - 설정 / 테마 / 팔레트
```

## 데이터 흐름

```text
[열기]
  UI(features/open-file) -> IPC open_file(path)
    -> Rust(src-tauri/commands) -> crates/fs-engine.read
    -> { text, encoding, mtime }
  -> entities/document.setDocument(text, meta)  (텍스트 = 진실의 원천)
  -> widgets/editor-pane -> EditorFacade.setDocument(text)

[편집]
  CM6 변경 -> EditorFacade.onChange -> entities/document (dirty=true, 통계 갱신)

[저장]
  UI(features/save-file) -> EditorFacade.getDocument() -> { text }
    -> IPC save_file(path, text) -> crates/fs-engine.atomicWrite (temp->rename)
    -> { mtime } -> entities/document (dirty=false)
```

## 세울 경계 (컴파일 가능한 계약)

```text
packages/core
  Document: { path, text, encoding, isDirty, mtime }
  createDocument / setText / markSaved  (순수, 테스트 대상)

packages/editor
  EditorFacade: mount / setDocument / getDocument / onChange / destroy
  (내부 CM6는 감춘다 — 앱은 이 인터페이스만 안다)

packages/ipc
  open_file / save_file 타입(인자·반환) 단일 출처

crates/fs-engine
  read(path) -> (text, encoding, mtime)
  atomic_write(path, text) -> mtime   (temp + fsync + rename)
  validate_path(root, path)           (탈출 방지)
```

## 검증 (완료 기준)

```text
단위:
  packages/core     문서 생성/편집/저장 전이(dirty true/false)
  crates/fs-engine  원자적 저장(크래시 시 원본 보존 시뮬), 인코딩 왕복, 경로 탈출 거부

통합:
  packages/editor   EditorFacade가 setDocument -> getDocument 왕복으로 텍스트 보존

E2E(Linux/Windows):
  프로토타입 이후 단계에서 연결

수동(macOS):
  앱 실행 -> 파일 열기 -> 텍스트 수정 -> 저장 -> 디스크 반영 확인

성능:
  최소 벤치 하니스가 같은 macOS 머신에서 콜드 스타트/메모리/입력 smoke 리포트 생성
```

## 이 슬라이스가 증명하는 것

- **경계가 옳다**: 앱이 CM6를 직접 모르고 facade로만 편집한다(교체 가능성).
- **원천이 하나다**: 텍스트가 유일한 진실의 원천이고 저장이 그 텍스트를 그대로 쓴다.
- **프로토타입이 실제 앱이다**: macOS에서 Tauri 앱 실행, 편집, 저장이 실제로 된다.
- **저장이 안전하다**: 원자적 저장으로 문서 손상 위험이 없다.
- **성능 측정 경로가 열렸다**: 이후 라이브 프리뷰와 Worker가 들어와도 회귀를 숫자로 볼 수 있다.

이 네 가지가 성립하면 이후 라이브 프리뷰·Worker·워크스페이스를 얹어도 구조가 흔들리지 않는다.
