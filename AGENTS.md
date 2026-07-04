# 에이전트 인덱스

이 문서는 이 프로젝트에서 작업하는 에이전트가 가장 먼저 읽는 진입점이다. 실제 규칙과 설명은 링크된 문서를 **단일 출처**로 둔다. 이 파일에는 규칙 본문을 중복해서 적지 않는다.

> 프로젝트 코드네임은 아직 미정이다. 문서 전반에서는 "이 에디터"로 지칭한다. 정식 이름이 정해지면 문서·패키지명을 함께 갱신한다.

## 한 줄 정의

웹 생태계(Tauri + React) 위에서 만드는 크로스 플랫폼(Windows/Linux/macOS) 마크다운 데스크탑 에디터. **원본 마크다운 서식을 보존하는 "마크다운을 다루는 도구"** 를 지향하며, Obsidian/Typora 대비 **시작 속도·메모리·바이너리 크기에서 체감 우위**를 목표로 한다.

## 먼저 읽을 문서

- [기술 스택](docs/tech-stack.md)
- [설계 의사결정 기록(ADR)](docs/decisions.md)
- [초기 아키텍처](docs/architecture.md)
- [필수 프로젝트 규칙](docs/project-rules.md)
- [개발 명령](docs/development-commands.md)
- [파일/폴더 구조](docs/project-structure.md)
- [PR 체크리스트](docs/pr-checklist.md)

## 설계 문서

- [에디터 전략(CodeMirror 6, 텍스트 원천, 라이브 프리뷰)](docs/editor-strategy.md)
- [마크다운 파이프라인(unified: remark/rehype, Web Worker)](docs/parser-pipeline.md)
- [프론트엔드 구조(Feature-Sliced Design)](docs/frontend-fsd.md)
- [Tauri 셸 경계(Rust 백엔드, IPC, 파일 I/O)](docs/tauri-shell.md)
- [성능 예산과 벤치 방법론](docs/performance-budget.md)
- [테스트 전략(단위·E2E·커버리지)](docs/testing-strategy.md)
- [검증 매트릭스](docs/verification-matrix.md)
- [실제 구현 계획](docs/implementation-plan.md)
- [초기 세로 슬라이스](docs/initial-vertical-slice.md)
- [레퍼런스와 clean-room 정책](docs/references.md)

## 핵심 원칙

작업자는 [필수 프로젝트 규칙](docs/project-rules.md)을 따라야 한다. 이 문서에 없는 결정이 필요하거나, 실제 구현이 설계 문서와 달라지는 경우에는 **임의로 진행하지 말고 사용자에게 먼저 보고**한다. 전략을 바꿔야 하면 코드보다 [설계 의사결정 기록](docs/decisions.md)을 먼저 갱신한다.
