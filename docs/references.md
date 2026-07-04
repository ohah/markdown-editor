# 레퍼런스와 clean-room 정책

이 에디터는 경쟁작·오픈소스를 **참고(레퍼런스/오라클)** 로만 쓰고, 소스를 복사하지 않는다. 구현은 공개 문서·명세·API에서 유도한다.

## 레퍼런스 용도

경쟁작과 라이브러리는 다음 용도로만 쓴다.

- 구조/UX 기준점(무엇이 좋은 편집 경험인가)
- 동작 비교 오라클(같은 마크다운 → 렌더 결과 비교)
- 성능 기준점(비교군, [performance-budget.md](performance-budget.md))

사용하지 않을 것:

- 경쟁작(Obsidian 등)의 비공개 구현을 리버스 엔지니어링해 그대로 옮기기
- copyleft(GPL/LGPL) 소스를 읽고 구조를 그대로 이식하기

## 경쟁작 (오라클 / 기준점)

| 이름 | 스택 | 참고 포인트 |
| --- | --- | --- |
| Obsidian | Electron + CM6 | 라이브 프리뷰 UX, 텍스트 원천 모델(우리와 같은 계열). 비공개. |
| Typora | Electron + 자체 렌더 | WYSIWYG UX(우리는 채택 안 함, ADR-003). |
| Logseq / Mark Text | Electron | 성능 비교군. |

경쟁작은 **동작 비교와 성능 기준점**으로만 쓴다. 특히 라이브 프리뷰 "느낌"은 참고하되, 구현은 CM6 decoration에서 독립 유도한다([editor-strategy.md](editor-strategy.md)).

## 핵심 라이브러리 (공식 문서 기준)

| 라이브러리 | 역할 | 근거 |
| --- | --- | --- |
| Tauri 2 | 데스크탑 셸 | 공식 문서·API. |
| CodeMirror 6 | 편집 엔진 | 공식 가이드(state/view/decoration/lang-markdown). |
| Lezer | 증분 마크다운 파서 | CM6 내장. |
| unified / remark / rehype | 리딩 뷰·익스포트 렌더 | 공식 플러그인 생태계. |
| React 19 | UI | 공식 문서. |
| Vite / Bun | 빌드·워크스페이스 | 공식 문서. |
| WebdriverIO / tauri-driver | E2E | 공식 문서(플랫폼 한계 포함, [testing-strategy.md](testing-strategy.md)). |

## 라이브 프리뷰 참고 시 주의

라이브 프리뷰는 오픈 구현(CM6 decoration 기반 예제 등)을 **패턴 참고**할 수 있으나, 라이선스를 확인하고 코드를 그대로 복사하지 않는다. 참고한 패턴의 출처와, 우리가 왜 그 방식을 택/변형했는지를 코드 주석이나 [decisions.md](decisions.md)에 남긴다.

## 표준·명세

- 마크다운: CommonMark 명세 + GFM(GitHub Flavored Markdown) 확장.
- 동작이 명세가 아니라 "사실상 표준"(도구마다 다른 부분)이면, 무엇을 베이스로 했고 왜 그 쪽을 택했는지 코드/문서/PR에 남긴다([project-rules.md](project-rules.md)의 근거 기록 원칙).

## 로컬 레퍼런스 체크아웃

경쟁작/라이브러리 소스를 로컬에서 읽어야 하면 `references/`(git 미커밋)에 두고 그 경로에서만 열람한다. copyleft 소스는 구현 유도용으로 열지 말고 최종 동작 비교(오라클)로만 쓴다.
