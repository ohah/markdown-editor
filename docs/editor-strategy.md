# 에디터 전략 (CodeMirror 6)

이 에디터의 편집 표면은 CodeMirror 6이다. 진실의 원천은 **마크다운 텍스트 문자열**이며([decisions.md ADR-003](decisions.md)), 소스 모드·라이브 프리뷰·리딩 뷰는 모두 이 하나의 텍스트에서 파생된다. 편집 엔진은 `packages/editor`에 가두고 앱에는 좁은 인터페이스만 노출한다(교체 가능성 유지).

## 왜 CM6인가

- 텍스트 원천을 유지하면서 증분 파싱(Lezer), 뷰포트 가상화(대용량 문서), decoration 시스템(라이브 프리뷰)을 모두 제공한다.
- contenteditable 위에서 브라우저가 **한글 IME 조합**을 처리한다(B/네이티브 대비 결정적 이점).
- 트랜잭션 기반 상태 모델이라 언두/리두, 협업 확장, 플러그인 경계가 깔끔하다.

## 세 가지 뷰 (하나의 원천)

```text
소스 모드      : 순수 마크다운. 문법 마크가 항상 보인다.
라이브 프리뷰  : 커서가 있는 줄은 마크 노출, 벗어난 줄은 인라인 렌더(굵게/기울임/링크/헤딩 등).
리딩 뷰        : 완전 렌더된 HTML(읽기전용). unified Worker가 생성([parser-pipeline.md](parser-pipeline.md)).
```

소스/라이브 프리뷰는 **같은 CM6 인스턴스**의 decoration 차이일 뿐이다. 리딩 뷰만 별도 렌더 경로다. 뷰 전환 시 원천 텍스트는 바뀌지 않으므로 왕복 손실이 없다.

## 라이브 프리뷰 구현 개요

라이브 프리뷰는 이 제품의 핵심 편집 경험이자 손이 가장 많이 가는 부분이다. CM6 `ViewPlugin` + `Decoration`으로 구현한다.

```text
1. Lezer markdown 파스 트리에서 노드(강조/링크/헤딩/코드/리스트 등)를 순회한다.
2. 커서/선택이 걸친 줄(또는 노드)은 "원문 노출" 상태로 둔다.
3. 커서가 벗어난 노드는:
   - 문법 마크(`**`, `#`, `[]()` 등)를 replace decoration으로 숨기고
   - 렌더된 형태(굵게/링크/헤딩 크기)를 mark/widget decoration으로 입힌다.
4. 선택 변경마다 최소 재계산(뷰포트 + 변경 노드 주변)만 수행한다.
```

원칙:

- **decoration은 순수 함수로**: 문서 + 선택 → decoration 집합. 부수효과 없이 계산해 테스트 가능하게 한다.
- **뷰포트 밖은 계산하지 않는다**: CM6 뷰포트만 처리해 대용량 문서에서 O(문서크기)를 피한다.
- **위젯(수식/mermaid/이미지)은 지연 렌더**: 무거운 위젯은 뷰포트 진입 시점에만 그린다.

## 확장 구성

```text
packages/editor/
  facade.ts        EditorFacade — 앱이 쓰는 좁은 인터페이스(생성/파괴/문서 교체/뷰 모드/이벤트)
  state.ts         EditorState 조립(문서, 히스토리, 키맵)
  markdown.ts      @codemirror/lang-markdown 설정(GFM 등)
  live-preview/    decoration 계산(순수) + ViewPlugin(적용)
  keymap.ts        편집 키맵(마크다운 단축: 굵게/링크/리스트 토글 등)
  theme.ts         에디터 테마(라이트/다크, config 연동)
```

## 앱에 노출하는 인터페이스(예시 계약)

```text
EditorFacade
  mount(container, initialDoc): void
  setDocument(text, meta): void          // 파일 열기
  getDocument(): { text, isDirty }       // 저장용
  setViewMode('source'|'live'|'reading'): void
  onChange(cb): Unsubscribe               // dirty/통계 갱신
  destroy(): void
```

앱은 이 인터페이스 뒤의 CM6 세부를 몰라야 한다. 이 경계가 [decisions.md ADR-002](decisions.md)의 "나중에 B로 교체" 되돌림 경로를 값싸게 유지한다.

## 성능 관점

- 편집은 메인 스레드에서 CM6가 담당한다. 라이브 프리뷰 decoration 계산이 hot path이므로, 선택 변경마다 **뷰포트+변경분만** 재계산한다.
- 리딩 뷰 전체 렌더는 절대 편집 스레드에서 하지 않는다(Worker).
- 회귀 감시: 라이브 프리뷰 on/off 상태의 입력 레이턴시를 [성능 예산](performance-budget.md) 벤치로 각각 측정한다.

## IME / 한글

- CM6/contenteditable의 composition 처리를 신뢰하되, **decoration이 조합 중(composition)에 원문 마크를 건드리지 않도록** 한다(조합 중 replace decoration이 커서 위치를 흔들면 마지막 자모 삭제·확정 버그가 난다).
- 조합 중인 줄은 항상 "원문 노출" 상태로 강제한다. 조합 종료(compositionend) 후에만 그 줄의 프리뷰를 재적용한다.
- 한글 입력 회귀는 E2E로 잡기 어려우므로 수동 검증 시나리오를 [테스트 전략](testing-strategy.md)에 남긴다.
